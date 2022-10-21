---
layout: post
title: "Overpass 2 - Hacked"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [wireshark, JTR, password-cracking, Source-code, hashcat, SUID]
image: ../../assets/img/posts/overpass2hacked.png 

---

## Description

Overpass has been hacked! Can you analyse the attacker's actions and hack back in?

|**Room**|[Overpass 2 Hacked](https://tryhackme.com/room/overpass2hacked)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[NinjaJc01](https://tryhackme.com/p/NinjaJc01)|

---

Let's begin the task by downloading the .pcap file attached with the task. 

Now, open the wireshark and start inspecting the file and there is a packet number 14 which is a POST request to **/development/upload.php** page,

![image](https://user-images.githubusercontent.com/67465230/186586477-e28a52b4-e291-412f-a88e-7dacf70daa4f.png)

Inspecting the packet by following its TCP stream,

![image](https://user-images.githubusercontent.com/67465230/186586509-de859b2d-39d1-46a1-886e-433815916eb4.png)

and we got that there is a netcat reverse shell one-liner executed in php.

Inspecting other packets,

![image](https://user-images.githubusercontent.com/67465230/186586546-1075152f-3437-4ca4-b91f-28b561034a4c.png)

Inspecting the packet while viewing the TCP stream,

![image](https://user-images.githubusercontent.com/67465230/186586581-6c6cc74c-f66c-4666-9333-e4515c6d3dcc.png)

we can see that there is a password of james user revealed in the packet.

Scrolling down the little bit, we can see that there are some sha512crypt hashes and a github link of ssh-backdoor by NinjaJc01,

![image](https://user-images.githubusercontent.com/67465230/186586645-fb2eb991-674a-4fad-b1e4-ceb7c209b34e.png)

From here, we create a file named hashes and put all the hashes in the file and using JTR, we can crack these hashes,

```bash
john hashes --wordlist=/usr/share/wordlists/fasttrack.txt --format=sha512crypt
```

![image](https://user-images.githubusercontent.com/67465230/186586700-0b596f60-a62f-468b-b575-589ea350cb4d.png)

we got couple of passwords for some users.

After this, let's move to analyze the source code of the ssh-backdoor written by NinjaJc01 which is available [here](https://github.com/NinjaJc01/ssh-backdoor) and we can see the default hash for the backdoor,

![image](https://user-images.githubusercontent.com/67465230/186586763-6955c1a5-246a-42fd-b2c3-399547cdba82.png)

Scrolling down and we can see the hardcoded salt for the backdoor,

![image](https://user-images.githubusercontent.com/67465230/186587609-2141fa3b-18cd-4ab7-b9ea-81b6788ff065.png)

Going back to pcap file and inspecting the packet and there is a hash that attacker has used,

![image](https://user-images.githubusercontent.com/67465230/186587641-7827b8a6-39e4-49f1-a06a-fd80d5563463.png)

Carefully taking a look at the source code of the ssh-backdoor and there is name of the hash, which is *sha512*,

![image](https://user-images.githubusercontent.com/67465230/186587665-c8139393-cb22-4dbf-99c5-4689dce39976.png)

Let's quickly find out mode of the hash,

```bash
hashcat -h | grep sha512
```

![image](https://user-images.githubusercontent.com/67465230/186587690-9cd88b5b-4370-4da2-aee8-47f7f9ba6463.png)

There are actually couple of modes for sha512 hash. 

Since we have a hash and a salt so we will use mode 1710 for `sha512($pass.$salt)` and using hashcat, we can crack the hash,

```bash
hashcat -m 1710 -a 0 task2_hash /usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/197255815-98b2ee68-8edd-4cc1-a82a-e3a8b5d74234.png)

Now that we have analyzed the pcap file, it's time to scan the machine for open ports,

```bash
rustscan -a 10.10.48.114 --range 0-65535 --ulimit 5000 -- -sVC -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/186587912-5777a6f5-86b6-4e47-9c92-b1b85425d1f8.png)

Result scan shows that port 22 is running SSH 7.6p1, port 80 is running apache webserver and port 2222 is also running SSH 8.2p1 service. 

Let's start enumerating port 80 by visiting http://$IP and we will be presented by this webpage where we can see the cactus!! 

![image](https://user-images.githubusercontent.com/67465230/186587978-013e12a9-71e3-48e6-9728-f992d150e711.png)

Next, before trying to enumerate other thing, I would like to try out the password for james user we cracked earlier. So I tried to SSH on port 22 but I failed and then I tried to SSH again on port 2222 and finally, I am now in the system,

```bash
ssh james@10.10.48.114 -p 2222
```

![image](https://user-images.githubusercontent.com/67465230/186588001-604eb1c4-1351-4c05-9a39-d027474cd29d.png)

Enumerating directory and there I find the user.txt flag and SUID binary,

![image](https://user-images.githubusercontent.com/67465230/186588030-422232cf-4ae5-47e5-aa0b-ec92ff33c88b.png)

SUID is a special type of file permissions given to a file. It allows the file to run with permissions of whoever the owner is. If this is root, it runs with root permissions.

Running this SUID binary with `-p` flag to preserve the privileges,

![image](https://user-images.githubusercontent.com/67465230/186588061-b34133d3-c491-4f73-b243-2b997c330575.png)

we become the root user. *Wuahahahahahaha....*