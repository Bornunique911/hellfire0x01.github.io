---
layout: post
title: "Chocolate Factory"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [rustscan, ftp, base64, hashcat, Command-Injection, id_rsa, vi, python]
image: ../../assets/img/posts/chocolatefactory.png 

---

## Description

A Charlie And The Chocolate Factory themed room, revisit Willy Wonka's chocolate factory!

|**Room**|[Chocolate Factory](https://tryhackme.com/room/chocolatefactory)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[AndyInfoSec](https://tryhackme.com/p/AndyInfoSec), [saharshtapi](https://tryhackme.com/p/saharshtapi), [0x9747](https://tryhackme.com/p/0x9747)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.113.243 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187381009-f9560e05-aaf2-4ca8-b466-2f488b46b780.png)

There are multiple open ports which we can scan them using nmap,

```bash
nmap -sC -sV 10.10.113.243 -p21,22,80,100-129 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187381064-364ac76e-5ae1-46da-a93b-e25ead10544b.png)

Scan results shows that port 21 is running FTP service, port 22 is running ssh service, port 80 is running apache webserver, port 100 is running unknown service.

Let's try to access ftp service,

```bash
ftp 10.10.113.243
```

![image](https://user-images.githubusercontent.com/67465230/187381117-9885ebeb-8263-4fae-9450-4fa1bb3b73f0.png)

we're now connected to the service. 

Enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187381162-8db0625a-5a0c-48b3-972b-a904407f5ae3.png)

there is one gum_room.jpg image in the directory.

We can download this image on our system,

```bash
get gum_room.jpg
```

Extract hidden data from image (steganography), 

```bash
steghide extract -sf gum_room.jpg
```

Reading the text file containing base64 string,

![image](https://user-images.githubusercontent.com/67465230/187381223-2f053f6d-323e-4d07-a7d2-5458105e212e.png)

Now, we can decode this base64 string,

```bash
echo "<base64-string>" | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/187381261-c0ca6b54-ed35-4f69-99b1-3dede5bdc91e.png)

This seems like a /etc/passwd file content. We have charlie user password hash.

We can copy this hash and paste this in file and using hashcat, a password cracking tool, to crack this password,

```bash
hashcat.exe -m 1800 crack.txt rockyou.txt -O
```

![image](https://user-images.githubusercontent.com/67465230/187381324-cce214a4-1bb8-4c03-a6ab-821572caacc7.png)

after sometime, we got the password cracked.

Visiting http://10.10.59.17 and we got a login page,

![image](https://user-images.githubusercontent.com/67465230/187381374-55cea6cd-d8a9-4599-9020-349d1b573bfa.png)

Using credentials we found, we can login into web application.

We got the webpage where we can execute commands,

![image](https://user-images.githubusercontent.com/67465230/187381440-ea7ebd5d-9b5a-4504-91e1-1b28cb53686c.png)

Let's execute `id` command,

![image](https://user-images.githubusercontent.com/67465230/187381502-51de3960-a604-436f-be60-396113d6e454.png)

we are **www-data** user.

Enumerate current directory using `ls`,

![image](https://user-images.githubusercontent.com/67465230/187381561-d6cc5b05-685a-4a11-8834-5045cb9c5353.png)

we got some files.

We can view some strings of binary,

```bash
strings key_rev_key
```

![image](https://user-images.githubusercontent.com/67465230/187381670-80bbbb48-dfae-411e-ba8d-e3e1f20de8f2.png)

Listing charlie user directory content,

![image](https://user-images.githubusercontent.com/67465230/187381744-708bd237-2f95-4128-a320-9d52c1fad495.png)

we got some files similar to id_rsa and user flag.

We can view the content of teleport file,

```bash
cat /home/charlie/teleport
```

![image](https://user-images.githubusercontent.com/67465230/187381841-f05ec8db-a3ef-4f82-9ace-5f6d11e45e90.png)

Let's copy the file content and paste it into teleport file and giving suitable file permission,

```bash
chmod 600 teleport
ssh -i teleport charlie@10.10.34.9
```

![image](https://user-images.githubusercontent.com/67465230/187381883-93fb6239-389b-4076-91dc-fc3a7254bc86.png)

Logging in using id_rsa file and we get into the system.

Enumerating home directory and we get our user flag,

![image](https://user-images.githubusercontent.com/67465230/187381933-a7ae18e8-0994-48b3-8fa6-ba2d7a9af47e.png)

**Privilege Escalation** : After getting system information, I decided to list all those binaries which we can run using `sudo`,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/187381988-3dc6d0db-957b-41b0-9362-b482ad71710a.png)

We can use **/usr/bin/vi** binary to escalate our privileges.

Using the command to get into editor,

```bash
sudo /usr/bin/vi
```

Now type `:!sh` in editor and we escape the restricted environment, hence we become root,

![image](https://user-images.githubusercontent.com/67465230/187382068-d6cb097c-6e3c-4ae2-ac6c-b8812331e682.png)

navigating to root directory and enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187382127-4801b27b-ffc8-4ec0-ac6b-0e09923cca42.png)

we have an unusual python file. 

Reading the python file content,

![image](https://user-images.githubusercontent.com/67465230/197122500-8bd34d1d-3ed1-4854-80d3-ac6646f33baf.png)

It seems like this is a script which has an encrypted message which can be only decrypted when key is provided. 

So, run the file and provide the key that we found earlier,

```bash
python root.py
```

![image](https://user-images.githubusercontent.com/67465230/187382292-ca27d3f3-3a3b-4f1a-ad8c-973da9b27bc1.png)

we got our root flag.