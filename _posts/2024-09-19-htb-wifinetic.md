---
layout: post
title: "HTB WifineticTwo"
tags: Wifi OpenWRT Wireless
---


![image](/images/16c5889acc1ca177c6b343c76bebcdaf.webp){:height="150px" style="float:right; margin: 10px" }
WifineticTwo is a Medium rated linux machine that presents an OpenPLC webserver, accessible through default credentials. This access can then be leveraged to perform remote code execution, granting initial access to the system. After establishing a foothold, a WPS Pixie Dust attack can be performed to obtain the Wi-Fi password for the associated Access point. Gaining access to the wireless network allows us to target the router running OpenWRT, where access to the web interface is used to obtain root privileges.
<!--more-->
## Recon
The first step of any HTB machine is always an nmap scan, allowing us to see what ports and services are open, and determine both the next steps in our enumeration and the likely vector of our attack:
```
└─$ nmap -sC -sV -p- --min-rate=1000 -T4 10.10.11.7
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-19 15:43 AEST
Nmap scan report for wifinetictwo.htb (10.10.11.7)
Host is up (0.062s latency).                                                                                                                                                                                      
Not shown: 65533 closed tcp ports (reset)                                                                
PORT     STATE SERVICE    VERSION                                                                        
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                                                                                           
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)            
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)           
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)         
8080/tcp open  http-proxy Werkzeug/1.0.1 Python/2.7.18                    
|_http-server-header: Werkzeug/1.0.1 Python/2.7.18                                                       
| http-title: Site doesn't have a title (text/html; charset=utf-8).       
|_Requested resource was http://wifinetictwo.htb:8080/login
```
We see two ports open; TCP port 22 for SSH (The version fingerprint indicating that this is an Ubuntu machine), and TCP port 8080 running a HTTP webserver with python Werkzeug. With only these two ports to go off of, our next logical step will be to explore the webserver.

## Getting a shell

Visiting the website at `10.10.11.7:8080` presents us with an openPLC login page. OpenPLC is a suite of tools for creating and running Programmable Logic Controllers, or PLCs.

![image](/images/WifineticTwo/openPLClogin.png){:style="margin:0 auto; display:block;"}

A quick google search lets us find this [page](https://autonomylogic.com/docs/2-1-openplc-runtime-overview/), which reveals that the default login credentials are `openplc : openplc`.
We submit these credentials through the login form and successfully authenticate, giving us access to the web interface:

![image](/images/WifineticTwo/openPLCinterface.png){:style="margin:0 auto; display:block;"}


Looking around the OpenPLC interface, we see a "Programs" section, which allows a user to upload, compile, and run C code. This seems like it could easily be used for remote code execution, so we take a quick look around online for existing exploits and find this [github PoC](https://github.com/thewhiteh4t/cve-2021-31630/blob/main/cve_2021_31630.py):

![image](/images/WifineticTwo/GithubExploit.png){:style="margin:0 auto; display:block;"}

The exploit works by uploading a reverse shell written in C to the "Programs" section of openplc, then compiling it and running it.
We clone the exploit, set up a netcat listener, and run it:

![image](/images/WifineticTwo/OpenPLCexploit.png){:style="margin:0 auto; display:block;"}

The exploit works! And since we caught a root shell, we're most likely running in a container, and we'll need to find our root flag on the host.
While we're here, we can grab the user.txt file from `/root/user.txt`.

## Privilege Escalation
We transfer linpeas to the machine and run it, confirming our suspicions of being contained:

![image](/images/WifineticTwo/linpeasContainer.png){:style="margin:0 auto; display:block;"}

Linpeas also reveals that we have a wireless interface on the machine, wlan0 (which we could've found ourselves by running `ip a`):

![image](/images/WifineticTwo/linpeasInterfaces.png){:style="margin:0 auto; display:block;"}

We run `iw dev wlan0 scan` to scan for wireless access points on the wlan0 interface:

![image](/images/WifineticTwo/iwdevscan.png){:style="margin:0 auto; display:block;"}


We see a device with a BSSID of `02:00:00:00:01:00` and an SSID of `plcrouter`. Interestingly, the output also shows that WPS is supported.
WPS stands for Wifi-Protected Setup, and it's vulnerable to both online and offline brute force attacks that enable an attacker to obtain the PSK, or pre-shared key.
WPS allows devices to join a Wi-Fi network without using the network's PSK, usually by using a push-button or a static eight-digit PIN code provided by the access point.

We'll use a python tool, OneShot.py, to perform a pixie dust attack on the access point. 
The pixie dust attack takes advantage of the way the nonces are generated for the WPS exchange; for the exchange to be secure, the nonces need to be random, but they are instead derived from the hash of the PSK. This allows us to figure out what the nonces are, and then use the rest of the information collected during the exchange to figure out the PSK.
After grabbing OneShot.py from https://github.com/AAlx0451/OneShot and transferring it to the machine, we can run the pixie dust attack:

![image](/images/WifineticTwo/OneShot.png){:style="margin:0 auto; display:block;"}

The attack works, and we get the PSK of `NoWWEDoKnowWhaTisReal123!`.

Now that we have the PSK for the AP, we can connect to it using our wireless interface.
First we need to generate a conf file using `wpa_passphrase`, and then we can connect using `wpa_supplicant`:

`wpa_passphrase plcrouter 'NoWWEDoKnowWhaTisReal123!' > wpa.conf`

`wpa_supplicant -B -i wlan0 -c ~/wpa.conf`

![image](/images/WifineticTwo/connectingToAP.png){:style="margin:0 auto; display:block;"}

Now that we're connected, we can run `arp -an` to view the arp cache and see the IP address of the router:

![image](/images/WifineticTwo/arp.png){:style="margin:0 auto; display:block;"}

The address is 192.168.1.1. We use netcat to do a quick scan of common ports and see what's open:

`nc -vz 192.168.1.1 21 22 80 443`

![image](/images/WifineticTwo/NetCatScan.png){:style="margin:0 auto; display:block;"}

We see that port 22, 80, and 443 are open. We use curl to take a quick look at the web page:

![image](/images/WifineticTwo/curl.png){:style="margin:0 auto; display:block;"}

In the curl response, we see `luCI Lua Configuration Interface`. A quick search online reveals that this is related to OpenWRT, an open source operating system designed for embedded devices to route network traffic. We can assume that this is what the router we connected to is running.

To get proper access to the webpage, we decide to use chisel to port forward port 80:

`sudo ./chisel server -p 8000 -reverse`

![image](/images/WifineticTwo/chiselServer.png){:style="margin:0 auto; display:block;"}


`./chisel client 10.10.14.15:8000 R:192.168.1.1:80 &`

![image](/images/WifineticTwo/chiselClient.png){:style="margin:0 auto; display:block;"}

Visiting localhost in our browser presents us with a login interface for luci, with a username of root already entered into the form:

![image](/images/WifineticTwo/luciLogin.png){:style="margin:0 auto; display:block;"}

Looking at the [OpenWRT documentation](https://openwrt.org/docs/guide-quick-start/walkthrough_login) reveals that the root user is configured to have no password set in a default installation, so we're able to login with the username `root` and no password.
Taking a brief look around the administration panel, we find something interesting in `System > Administration`. Under the SSH access tab, we see that ssh access is enabled, and root logins with password are also enabled. 

![image](/images/WifineticTwo/sshAccessOpenWRT.png){:style="margin:0 auto; display:block;"}

Considering that documentation revealed that there is no password set for root by default, this could mean that we're able to ssh in to the router as root with no password. We give it a try from our compromised container:

`ssh root@192.168.1.1`

![image](/images/WifineticTwo/SSHandROOT.png){:style="margin:0 auto; display:block;"}

We successfully get into the router, letting us grab the root flag and complete the box :).
