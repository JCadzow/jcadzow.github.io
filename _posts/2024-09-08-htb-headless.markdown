---
layout: post
title: HTB Headless
date: 2024-09-08 14:21:00
---
This is my first CTF writeup! I'm thinking of trying to do these regularly from now on, to both practice my writing skills and extract a bit of extra learning value out of each box I do; I figured that spending some extra time doing writeups would assist in remembering the details of boxes I've pwned.

Headless is an Easy difficulty linux machine that features a Python Werkzeug server hosting a website. The website has a support form, which we find is vulnerable to an XXS attack that we can use to steal an admin cookie, which gives us access to a dashboard that is vulnerable to command injection. After getting a shell on the box, we find that the user can run a particular shell script as sudo which doesn't use absolute paths, which we can exploit to get root.

