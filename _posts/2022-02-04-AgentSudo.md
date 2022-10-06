---
layout: post
title: "Agent Sudo"
date: 2022-02-04
categories: [Tryhackme, Easy-THM]
tags: [enumerate, exploit, brute-force, hash cracking]
image: ../../assets/img/posts/agentsudo.png 

---

## Description

You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth. 

|**Room**|[Agent Sudo](https://tryhackme.com/room/agentsudoctf)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[DesKel](https://tryhackme.com/p/DesKel)|

---

Let's deploy the machine and quickly start scanning ports with rustscan,

```bash
rustscan -a 10.10.115.44
```

![image](https://user-images.githubusercontent.com/67465230/187148779-77b18ab1-099a-457f-8a2d-bc0dbf99199c.png)

we got 3 open ports. Let's quickly scan them in detail using nmap.

```bash
nmap -sV -sC -p21,22,80 10.10.72.169 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187148828-826e0b92-2bff-4536-b9ac-f30a6ce9418b.png)

Scan result shows that port 21 is running vsftpd service (ftp), port 22 is running ssh and port 80 is running webserver. Let's enumerate port 80.

Visiting http://10.10.115.44,

![image](https://user-images.githubusercontent.com/67465230/187148889-5606e309-ebde-432f-b24e-b2f1e6cc2b6e.png)

we got a message from Agent R. Here, we have to use our codename as User-Agent to access the hidden page (which can't be find even with dirsearch).

>The **User-Agent** request header is a characteristic string that lets servers and network peers identify the application, operating system, vendor, and/or version of the requesting user agent.

To check where is this User-Agent header appears,

![image](https://user-images.githubusercontent.com/67465230/187148945-5441c160-1ebd-498f-9c49-880f629ecb87.png)

start burp and let it intercept the request made by this page and there we can see the on the 3rd line, the User-Agent request header. 

We can use "User Agent Switcher" extension to access that hidden page and we just have to Replace the User-Agent value field with our codename, **C**,

![image](https://user-images.githubusercontent.com/67465230/187149005-c5e6df9e-fdb9-4db3-9b73-66334c6fef51.png)

and clicking on **Apply (active window)** button.

After refreshing the page,

![image](https://user-images.githubusercontent.com/67465230/187149100-059bd764-e5f1-408f-ab1c-ae110e38b796.png)

we got the hidden page and we can see the new message. Also, we got the username chris. Username enumeration!

As we have username, we can still brute force the password for ftp service,

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.10.72.169
```

![image](https://user-images.githubusercontent.com/67465230/194365568-6d6e064e-00bf-46c2-8339-c7053f36b86c.png)

after sometime, we got our password `*******`.

Let's access the ftp service, 

```bash
ftp 10.10.72.169
```

![image](https://user-images.githubusercontent.com/67465230/187149238-fc4fd901-c679-43d6-afa2-8bac8fab2ad3.png)

We have now access to ftp service.

Enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187149316-4b7084fd-2c20-4258-9b21-3b6f5d42c46b.png)

we got 3 files in this directory. 

Let's download all these files,

```bash
mget *
```

with this command, we can download all files. 

Let's read the content of To_agentJ.txt file,

![image](https://user-images.githubusercontent.com/67465230/187149367-2499cf09-ff83-4aee-9980-45e179b3cb81.png)

After reading this message, we get the idea that the data is hidden within images we just downloaded. We can get the password which is stored as hidden data.

I used the steghide and zsteg tool to pull out the hidden data but I was unsuccessful to do so. I instead used **binwalk** tool (which can be downloaded using `sudo apt install binwalk`),

```bash
binwalk cutie.png
```

![image](https://user-images.githubusercontent.com/67465230/187149415-d44a941b-cb0d-41cf-aae5-062c93582939.png)

this command will display the hidden data (if present) inside the image, and it is in this image.

Let's extract this data,

```bash
binwalk -e cutie.png
```

![image](https://user-images.githubusercontent.com/67465230/187149472-5240a4d5-5a7a-4a13-8afd-e16a21eb5b81.png)

this command will extract all the information that is hidden and make a separate directory to store that extracted data.

Let's navigate to directory and list directory content,

![image](https://user-images.githubusercontent.com/67465230/187149541-4340e640-b98b-4ec9-a191-7ee914583051.png)

we got a zip file and a txt file but txt file is empty so it is not useful right now. We will focus on zip file.

While opening file with `unzip 8702.zip` command, it doesn't decompressed because we are lacking in providing password. And I don't know what the password is, so we can't extract the zip file content. But, we can crack this zip file password. Let's see how.

Using tool called **zip2john**, we will convert this zip file into crackable hash,

```bash
zip2john 8702.zip > crackme
```

![image](https://user-images.githubusercontent.com/67465230/187149592-b822120b-2e64-4c01-aed5-fe054cb51d49.png)

Now, since we have the hash, let's crack it,

```bash
john crackme
```

![image](https://user-images.githubusercontent.com/67465230/187149697-d5a632e7-0bdc-4d4e-9de1-9d49ead87c75.png)

we got the password **alien**.

Now, using built-in tool **7z**, we can extract the contents of zip file,

```bash
7z x 8702.zip
```

![image](https://user-images.githubusercontent.com/67465230/187149753-163e8f12-d78f-4886-87cc-f7c4ff25bdc7.png)

after pressing enter, we have to provide the password we just cracked and it will create the To_agentR.txt file.

Let's view the content of To_agentR.txt file,

![image](https://user-images.githubusercontent.com/67465230/187149798-598c556e-33f6-46a7-b353-1858dce7644b.png)

we got the unknown screen. Throwing this string on google reveals that this is a base64 string. 

Decoding this string,

```bash
echo QXJlYTUx | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/187149846-018c1988-d842-421f-ab31-4267c15ffa6e.png)

we got the password **Area51**.

Now, using password we got after decoding the string, we can extract the content of another image,

```bash
steghide extract -sf cute-alien.jpg
```

![image](https://user-images.githubusercontent.com/67465230/187149880-7a5d3cb7-17b4-49e4-ba05-824be40e3dad.png)

message.txt file gets extracted.

Let's read it's content,

![image](https://user-images.githubusercontent.com/67465230/187149952-00cb0991-ee1a-42af-8b44-b94f010fa204.png)

we got the username **james** & password **hackerrules!**.

Now, let's get in touch with the system,

```bash
ssh james@10.10.183.139
```

![image](https://user-images.githubusercontent.com/67465230/187150003-79a2c5a1-d66a-4ac0-987d-75608d5c4b94.png)

Authentication success. We are james user and we can confirm this using `whoami` command.

List directory content,

![image](https://user-images.githubusercontent.com/67465230/187150123-f8fd6791-3df2-4485-b675-37bde31b1c97.png)

we got the user flag.

There is a image file in the directory as well, so we have to transfer this file on our local system. I used **scp** tool. On my local vm, I run this command,

```bash
scp james@10.10.183.139:/home/james/Alien_autospy.jpg .
```

![image](https://user-images.githubusercontent.com/67465230/187150236-77d4fba8-dfd4-45b7-8721-ebdf14a64e02.png)

the image gets transferred.

Doing the Reverse google search to find out the image context,

![image](https://user-images.githubusercontent.com/67465230/187150302-dc01e3cf-0496-4876-a4f7-d3ca37f43370.png)

This image is the Roswell alien autopsy,

![image](https://user-images.githubusercontent.com/67465230/187150349-4a0ef85d-95cf-41d7-b0d8-6ff6aced388d.png)

Now comes the privilege escalation part. Let's find all those binaries which can be run as `sudo`,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/187150432-16a47239-aa59-44ec-a6d9-d47a66cd8b3a.png)

we can run `/bin/bash` binary with sudo to escalate.

Upon firing the command `sudo /bin/bash`, it didn't elevate our privilege (which should work),

![image](https://user-images.githubusercontent.com/67465230/187150546-b59c2b19-cf7c-4c6b-989f-f231d378869b.png)

as the message shows that "james user is not allowed to run '/bin/bash' as root user", means we have to think of other method to escalate our privileges.

After researching about this vulnerability, I came across the [CVE-2019-14287](https://blog.aquasec.com/cve-2019-14287-sudo-linux-vulnerability),

![image](https://user-images.githubusercontent.com/67465230/187150617-f0e51874-7cc7-4613-8136-51f3cdc2836d.png)

> **CVE-2019-14287** : The sudo vulnerability is a security policy bypass issue that provides a user or a program the ability to execute commands as root on a Linux system when the "sudoers configuration" explicitly disallows the root access. Exploiting the vulnerability requires the user to have sudo privileges that allow them to run commands with an arbitrary user ID, except root.

This vulnerability allows bypass of user restrictions means we can bypass the restrictions using other user id (for eg, id=0, which is id of user root).

Running the command,

```bash
sudo -u#0 /bin/bash
```

![image](https://user-images.githubusercontent.com/67465230/187150702-ee221a8c-8a46-43df-9a0b-e655b354fb55.png)

we will get the system access.
