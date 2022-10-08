---
layout: post
title: "Inclusion"
date: 2022-10-08
categories: [Tryhackme, Easy-THM]
tags: [LFI, /etc/passwd, socat, GTFObins]
image: ../../assets/img/posts/inclusion.png 

---

## Description

A beginner level LFI challenge.

|**Room**|[Inclusion](https://tryhackme.com/room/inclusion)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[falconfeast](https://tryhackme.com/p/falconfeast)|

---

Let's start with rustscan to quickly finds all open ports,

```bash
rustscan -a 10.10.181.84
```

![image](https://user-images.githubusercontent.com/67465230/173224804-087ca2e0-2f6c-452e-82fe-ddf95540e374.png)

We got 2 ports open. Let's run detail nmap scan on these open ports.

```bash
nmap -sC -sV -p22,80 10.10.181.84
```

![image](https://user-images.githubusercontent.com/67465230/173224809-0b62b38e-9d1e-4bfe-8ef0-964ef1e4546d.png)

We can see that port 22 is running SSH and port 80 is running webserver. Let's start enumeration of port 80.

Visit http://10.10.181.84,

![image](https://user-images.githubusercontent.com/67465230/173224815-8a014d76-0277-480e-b695-6fc2e8e4c40a.png)

and we'll be presented with this blog site. After enumerating, I found that LFI-attack section is vulnerable to LFI [Local File Inclusion](https://en.wikipedia.org/wiki/File_inclusion_vulnerability).

Let's try to open this LFI-attack section from blog post,

![image](https://user-images.githubusercontent.com/67465230/173224821-d6391b32-24cd-426e-b433-9b274a1fa598.png)

we got a little description about LFI vulnerability. 

Let's try to edit the article parameter in order to load content of /etc/passwd on webpage by 

```
?article=../../../../../../../../../etc/passwd
```

![image](https://user-images.githubusercontent.com/67465230/173224826-5a2b61fd-d68b-4f27-8c6b-0fe420b4145b.png)

and there we go. Now, we've content of /etc/passwd file. We now know that there is also a user called falconfeast.

But, let's take a clear look at at it's source code,

![image](https://user-images.githubusercontent.com/67465230/194687485-c97be089-6391-4deb-bbbd-43148fed754d.png)

looking at the source code reveals us the credentials of falconfeast user. Let's try these on SSH.

```bash
ssh falconfeast@10.10.181.84
```

![image](https://user-images.githubusercontent.com/67465230/186206218-c53f77b4-eb03-45db-b967-b52c26923f44.png)

Providing password will get us authenticated. We're **falconfeast** user and we can confirm this using `whoami` command.

Establish directory content using `ls -la`,

![image](https://user-images.githubusercontent.com/67465230/173224849-9429c8f9-6cd7-4e0f-9ed3-ac42517da4e6.png)

we got our user flag. 

Now, let's do privilege escalation in order to become root user. We'll search for those binaries which can be run as `sudo` without providing the password.

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/173224860-28fc2a6e-6d2d-4546-8e1c-dbfd5b059d83.png)

There is a binary **/usr/bin/socat** which can be run as sudo without providing password (we can become root).

Let's visit [GTFObins](https://gtfobins.github.io/) and search for socat,

![image](https://user-images.githubusercontent.com/67465230/173224892-91afd9b5-f0d6-4653-bc3f-f76aa7e4a85c.png)

we can see that if we run this command, we'll get system access (root).

Let's modify the command and execute,

```bash
sudo /usr/bin/socat stdin exec:/bin/bash
```

![image](https://user-images.githubusercontent.com/67465230/173224898-38330108-8052-46e4-a5a1-7153641c9a20.png)

We're now **root** user and confirm this using `id` command.