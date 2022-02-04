---
layout: post
title: "THM - Agent Sudo"
date: 2022-02-04
categories: [Tryhackme, Linux]
tags: [enumerate, exploit, brute-force, hash cracking]
image: ../../assets/img/posts/AgentSudo/agentsudo.png 

---

## Description

You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth. 

|**Room**|Agent Sudo|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Room Link**| https://tryhackme.com/room/agentsudoctf |
|**Creator**|[DesKel](https://tryhackme.com/p/DesKel)|

---

Let's deploy the machine and quickly start scanning ports with rustscan,

```bash
rustscan -a $IP
```

![rustscan](../../assets/img/posts/AgentSudo/rustscan.png)

we got 3 open ports. Let's quickly scan them in detail using nmap.

```bash
nmap -sV -sC -p21,22,80 $IP -oN nmap.txt
```

![nmap](../../assets/img/posts/AgentSudo/nmap.png)

Scan result shows that port 21 is running vsftpd service (ftp), port 22 is running ssh and port 80 is running webserver. Let's enumerate port 80.

Visiting http://$IP,

![web1](../../assets/img/posts/AgentSudo/web1.png)

we got a message from Agent R. Here, we have to use our codename as User-Agent to access the hidden page (which can't be find even with dirsearch).

>The **User-Agent** request header is a characteristic string that lets servers and network peers identify the application, operating system, vendor, and/or version of the requesting user agent.

To check where is this User-Agent header appears,

![burp](../../assets/img/posts/AgentSudo/burp.png)

start burp and let it intercept the request made by this page and there we can see the on the 3rd line, the User-Agent request header.

We can use "User Agent Switcher" extension to access that hidden page and we just have to Replace the User-Agent value field with our codename, C,

![agentswitcher](../../assets/img/posts/AgentSudo/agentswitcher.png)

and clicking on Apply (active window) button.

After refreshing the page,

![web2](../../assets/img/posts/AgentSudo/web2.png)

we got the hidden page and we can see the new message. Also, we got the username chris. Username enumeration!

As we have username, we can still brute force the password for ftp service,

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://$IP
```

![hydra](../../assets/img/posts/AgentSudo/hydra.png)

after sometime, we got our password `*******`.

Let's access the ftp service,

```bash
ftp $IP
```

![ftp1](../../assets/img/posts/AgentSudo/ftp1.png)

We have now access to ftp service.

Enumerating directory,

```bash
ls -la
```

![ftp2](../../assets/img/posts/AgentSudo/ftp2.png)

we got 3 files in this directory.

Let's download all these files,

```bash
mget *
```

![ftp3](../../assets/img/posts/AgentSudo/ftp3.png)

with this command, we can download all files.

Let's read the content of To_agentJ.txt file,

![msg1](../../assets/img/posts/AgentSudo/msg1.png)

After reading this message, we get the idea that the data is hidden within images we just downloaded. We can get the password which is stored as hidden data.

I used the steghide and zsteg tool to pull out the hidden data but I was unsuccessful to do so. I instead used binwalk tool (which can be downloaded using `sudo apt install binwalk`),

```bash
binwalk cutie.png
```

![binwalk1](../../assets/img/posts/AgentSudo/binwalk1.png)

this command will display the hidden data (if present) inside the image, and it is in this image.

Let's extract this data,

```bash
binwalk -e cutie.png
```

![binwalk2](../../assets/img/posts/AgentSudo/binwalk2.png)

this command will extract all the information that is hidden and make a separate directory to store that extracted data.

Let's navigate to directory and list directory content,

![ls](../../assets/img/posts/AgentSudo/ls.png)

we got a zip file and a txt file but txt file is empty so it is not useful right now. We will focus on zip file.

While opening file with `unzip 8702.zip` command, it doesn't decompressed because we are lacking in providing password. And I don't know what the password is, so we can't extract the zip file content. But, we can crack this zip file password. Let's see how.

Using tool called **zip2john**, we will convert this zip file into crackable hash,

```bash
zip2john 8702.zip > crackme
```

![zip2john](../../assets/img/posts/AgentSudo/zip2john.png)

Now, since we have the hash, let's crack it,

```bash
john crackme
```

![john](../../assets/img/posts/AgentSudo/john.png)

we got the password `*****`.

Now, using built-in tool **7z**, we can extract the contents of zip file,

```bash
7z x 8702.zip
```

![7z](../../assets/img/posts/AgentSudo/7z.png)

after pressing enter, we have to provide the password we just cracked and it will create the **To_agentR.txt** file.

Let's view the content of To_agentR.txt file,

![msg2](../../assets/img/posts/AgentSudo/msg2.png)

we got the unknown screen. Throwing this string on google reveals that this is a base64 string.

Decoding this string,

```bash
echo ******** | base64 -d
```

![base64](../../assets/img/posts/AgentSudo/base64.png)

we got the password `******`.

Now, using password we got after decoding the string, we can extract the content of another image,

```bash
steghide extract -sf cute-alien.jpg
```

![steghide](../../assets/img/posts/AgentSudo/steghide.png)

message.txt file gets extracted.

Let's read it's content,

![msg3](../../assets/img/posts/AgentSudo/msg3.png)

we got the username james & password `************`.

Now, let's get in touch with the system,

```bash
ssh james@$IP
```

![ssh](../../assets/img/posts/AgentSudo/ssh.png)

Authentication success. We are james user and we can confirm this using whoami command.

List directory content using `ls -la`,

![ssh2](../../assets/img/posts/AgentSudo/ssh2.png)

we got the user flag.

There is a image file in the directory as well, so we have to transfer this file on our local system. I used scp tool. On my local vm, I run this command,

```bash
scp james@$IP:/home/james/Alien_autospy.jpg .
```

![scp](../../assets/img/posts/AgentSudo/scp.png)

the image gets transferred.

Doing the Reverse google search to find out the image context,

![web3](../../assets/img/posts/AgentSudo/web3.png)

This image is the Roswell alien autopsy,

![web4](../../assets/img/posts/AgentSudo/web4.png)

Now comes the privilege escalation part. Let's find all those binaries which can be run as sudo,

```bash
sudo -l
```

![ssh3](../../assets/img/posts/AgentSudo/ssh3.png)

we can run /bin/bash binary with sudo to escalate.

Upon firing the command `sudo /bin/bash`, it didn't elevate our privilege (which should work),

![ssh4](../../assets/img/posts/AgentSudo/ssh4.png)

as the message shows that "james user is not allowed to run '/bin/bash' as root user", means we have to think of other method to escalate our privileges.

After researching about this vulnerability, I came across [CVE-2019-14287](https://blog.aquasec.com/cve-2019-14287-sudo-linux-vulnerability)

![web5](../../assets/img/posts/AgentSudo/web5.png)

> CVE-2019-14287 : The sudo vulnerability is a security policy bypass issue that provides a user or a program the ability to execute commands as root on a Linux system when the "sudoers configuration" explicitly disallows the root access. Exploiting the vulnerability requires the user to have sudo privileges that allow them to run commands with an arbitrary user ID, except root.

This vulnerability allows bypass of user restrictions means we can bypass the restrictions using other user id (for eg, id=0, which is id of user root).

Running the command, 

```bash
sudo -u#0 /bin/bash
id
```

![ssh5](../../assets/img/posts/AgentSudo/ssh5.png)

we will get the root access, and hence the root flag. (:
