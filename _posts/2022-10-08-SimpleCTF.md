---
layout: post
title: "SimpleCTF"
date: 2022-10-08
categories: [Tryhackme, Easy-THM]
tags: [rustscan, ftp, dirsearch, CMS-Made-Simple, SQL-Injection, vim]
image: ../../assets/img/posts/simplectf.png 

---

## Description

Beginner level ctf.

|**Room**|[SimpleCTF](https://tryhackme.com/room/easyctf)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[MrSeth6797](https://tryhackme.com/p/MrSeth6797)|

---

Let's deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.161.69
```

![image](https://user-images.githubusercontent.com/67465230/186826244-9421bb53-1c2b-434c-bc1a-4baca9a339d6.png)

we got 3 ports open. Let's scan them in detail with nmap.

```bash
nmap -sC -sV -p21,80,2222 10.10.161.69 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/186826266-847c1c3a-bc86-4389-8572-4b7b05096593.png)

Scan results reveals that port 21 is running ftp service with anonymous login, port 80 is running webserver and port 2222 is running ssh service (unusual). 

Let's enumerate port 21 by accessing the FTP service,

```bash
ftp 10.10.161.69
```

![image](https://user-images.githubusercontent.com/67465230/186826323-f85bd0a5-826a-4646-a1c2-33d41e995cd0.png)

and we get in.

Let's list directory content, 

![image](https://user-images.githubusercontent.com/67465230/186826363-4596e337-1d9b-455c-9cb4-810dee2b1357.png)

**pub** named directory is found.

Navigate to pub directory and list directory content,

![image](https://user-images.githubusercontent.com/67465230/186826390-f474eba2-e280-4ec1-ae29-71021915f22b.png)

we got a file named ForMitch.txt. This might be the username enumeration. 

Let's download this file on our system,

```bash
get ForMitch.txt
```

we have successfully downloaded this file.

Let's view this file content using `cat`,

![image](https://user-images.githubusercontent.com/67465230/186826469-b5dfc239-bf31-46cc-8901-f710bd81cc54.png)

Seems like Password can be crackable and is same for System User (very bad idea).

Let's explore port 80 by visiting http://10.10.169.69,

![image](https://user-images.githubusercontent.com/67465230/186826502-6b1d0183-2972-4bdc-8ad6-f0839e196863.png)

we land on a default ubuntu page.

At this point, we don't get anything useful from this webpage, so let's find hidden directories using dirsearch,

```bash
dirsearch -u http://10.10.169.69 -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301
```

![image](https://user-images.githubusercontent.com/67465230/186826524-1fdd9ede-0a3d-4289-812f-4d76fca3ce06.png)

we got a hidden directory named **simple**.

Let's check if there is something on robots.txt,

![image](https://user-images.githubusercontent.com/67465230/186826647-437ffa63-4e8c-40f8-bb4a-318e47656228.png)

we don't anything useful. There is a /openemr path but on visiting, it throws error so it is not useful.

Let's explore http://10.10.169.69/simple,

![image](https://user-images.githubusercontent.com/67465230/186826690-b3b83ac0-9b11-45c4-a73d-32a13a802daf.png)

we got a webpage made from CMS Simple. This page can be interesting. Let's explore.

Upon scrolling down, 

![image](https://user-images.githubusercontent.com/67465230/186826710-7029ee78-4fdd-4e49-9034-de0e50099e17.png)

we found the software name CMS Made Simple and it's version 2.2.8. Sweet. 

Let's quickly search for this software on google [cms 2.2.8 exploit](https://www.exploit-db.com/exploits/46635)

![image](https://user-images.githubusercontent.com/67465230/186826730-8dac939a-6584-47c5-8b77-e5decaf46fd8.png)

this is a sqli exploit. Download it so that we can run this exploit.

Let's try to use it,

```bash
python3 46635.py
```

![image](https://user-images.githubusercontent.com/67465230/186826762-0e7a9d70-2f7b-4574-ad90-1a5801db7fa1.png)

we can see the usage of this exploit.

Let's run the exploit with this command,

```bash
python3 46635.py -u http://10.10.169.69/simple --crack -w /usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/194691027-37751688-af36-4160-b3e6-f439d0735f76.png)

after sometime, we got the username and cracked password **secret**.

Now that we have credentials, why don't we try to login on ssh service, (remember, SSH service is running on port 2222) 

```bash
ssh mitch@10.10.169.69 -p 2222
```

![image](https://user-images.githubusercontent.com/67465230/186827090-0b42258d-33bb-4ede-9182-c487417b19e5.png)

and we got in.

Since we don't have a tty shell, we will now improve this shell,

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![image](https://user-images.githubusercontent.com/67465230/186827139-f40ddbb4-d94d-4601-bd23-9fb1c25defbf.png)

we get a friendly bash prompt. Let's enumerate directory using `ls -la` and we got our user.txt flag. Great.

Now, going back one directory,

![image](https://user-images.githubusercontent.com/67465230/186827170-3407039d-27ee-44d3-98b0-12ea2ebe3279.png)

we found that there is another user, **sunbath**. Maybe we can do a lateral movement and then escalate our privileges but it isn't the case (I found nothing after enumeration that can help us to achieve this).

Let's check what binaries we can run as sudo without providing password,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/186827201-6cfc31c7-f7f1-4eb1-8290-1b58abd70f35.png)

**/usr/bin/vim** binary can be run with sudo. 

Let's run this binary,

```bash
sudo /usr/bin/vim
```

![image](https://user-images.githubusercontent.com/67465230/186827275-4262f037-5b26-4110-8e95-c7ed808e45c6.png)

and we got in vim editor mode. Type **:** followed by **!bash** and then press enter,

and we got escaped from restricted environment to root user,

![image](https://user-images.githubusercontent.com/67465230/186827308-3dcbbb3c-39dd-458f-87b1-9f886a1d52c5.png)

we have *uid=0* means we are **root** and we can check this using `id` command. Now, we can navigate to root directory and grab the root.txt flag.
