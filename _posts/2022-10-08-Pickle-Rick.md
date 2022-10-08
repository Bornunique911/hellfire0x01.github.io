---
layout: post
title: "Pickle Rick"
date: 2022-10-08
categories: [Tryhackme, Easy-THM]
tags: [nmap, gobuster, base64, WebShell, Command-Injection]
image: ../../assets/img/posts/picklerick.png 

---

## Description

A Rick and Morty CTF. Help turn Rick back into a human!

|**Room**|[Pickle Rick](https://tryhackme.com/room/picklerick)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[tryhackme](https://tryhackme.com/p/tryhackme)|

---

Start and deploy the machine and let's scan the ports with nmap,

```bash
nmap -sV 10.10.3.4
```

![image](https://user-images.githubusercontent.com/67465230/186668399-4a10ba7f-61c9-468c-a9c2-48fbe78275ba.png)

We can see that 2 ports are open, i.e. 22 (SSH) and 80 (HTTP). Let's enumerate port 80 by visiting http://10.10.3.4,

We'll land on index of the box. 

![image](https://user-images.githubusercontent.com/67465230/186668464-295831eb-af52-4300-8476-938fc90a2d0a.png)

Let's enumerate it a bit.

There's a username on source page of the webpage,

![image](https://user-images.githubusercontent.com/67465230/186668551-77bcf7b9-ef5d-4880-b257-bc9a2b0ebfe5.png)

Username:R1ckRul3s.

Let's bruteforce the directories to find anything useful,

```bash
gobuster dir -u http://10.10.3.4 -w /usr/share/seclists/Discovery/Web-Content/common.txt -x txt,php -q 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186668617-334dbc28-ac4b-4865-8cb9-2d03a962cf88.png)

We got some interesting things. Let's start with robots.txt file,

![image](https://user-images.githubusercontent.com/67465230/194690185-2de1ea06-76eb-4bf5-b82c-b74334ff1135.png)

There's a strange string present. Maybe this is the password? Let's see :).

Then we'll navigate to /login.php page,

![image](https://user-images.githubusercontent.com/67465230/186668839-bd6d3eda-9ecc-403a-ba81-c9077c80933b.png)

seems like we've to have username and password in order to login.

Let's use the username and passwords we found on before,

![image](https://user-images.githubusercontent.com/67465230/186668886-876526e1-b4c2-4943-8de0-395db2cefeb5.png)

We get logged in.

Again enumerating it's source page, we got a string (maybe base64),

![image](https://user-images.githubusercontent.com/67465230/194690208-e55848d8-90ec-4d60-b4ac-32a50479c328.png)

After decoding it, I got another string and after that a new string, so I thought I have to decode it many times more in order to get a plain text string,

![image](https://user-images.githubusercontent.com/67465230/194690243-14199c8d-b105-4bfc-b00b-64504d37bcbb.png)

After decoding it many times, we got a string "Rabbit Hole".

Now, back to command panel, let's try to inject a command and see what happens,

```
id
```

![image](https://user-images.githubusercontent.com/67465230/186668931-ba95b919-5135-46db-a060-c65906301f9b.png)

We're www-data user. This webpage is vulnerable to OS Command Execution. 

```
Command Injection: Command injection is an attack in which the goal is execution of arbitrary commands on the host operating system via a vulnerable application. Command injection attacks are possible when an application passes unsafe user supplied data (forms, cookies, HTTP headers etc.) to a system shell. In this attack, the attacker- supplied operating system commands are usually executed with the privileges of the vulnerable application. Command injection attacks are possible largely due to insufficient input validation.
```

So now, we know that we can execute arbitrary commands on this machine to even read files (if possible).

Let's establish directory content,

```
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/186668983-105fafd8-7080-4280-993d-dea555623109.png)

There's a **Sup3rS3cretPickl3Ingred.txt** file, let's view it's content using cat command,

```
cat Sup3rS3cretPickl3Ingred.txt
```

![image](https://user-images.githubusercontent.com/67465230/186669037-fb1c29ee-1fce-42ef-b542-335f8dee8d4c.png)

Whoops.. we can't see the content of this file.

Let's try to read this file using **less** command,

```
ls -la; less Sup3rS3cretPickl3Ingred.txt
```

![image](https://user-images.githubusercontent.com/67465230/194690268-ea7b452a-590a-4dd5-8129-70c4e97fa70f.png)

We got our first Ingredient!! 

There's also a **clue.txt** file. Let's look at it's content,

```
less clue.txt
```

![image](https://user-images.githubusercontent.com/67465230/186669145-7b79ecc8-13e9-4de7-b7d9-2f8fcdbc9ef6.png)

It says "Look around the file system for other ingredient". Seems like we've to find other ingredients just like this.

Let's look for second one. We know that we an execute commands so we can traverse directories and establish content of system path /,

```
cd ../../../../../../../; ls -la
```

![image](https://user-images.githubusercontent.com/67465230/186669230-3a63a373-8fb4-4323-bab2-ccada1cdb589.png)

We got all content of the path. There's a /home directory which is worth checking.

```
cd /home; pwd; ls -la
```

![image](https://user-images.githubusercontent.com/67465230/186669261-ac9a16c4-3efb-400f-9dec-9a7d74e252f2.png)

We got 2 directories. "rick" directory seems interesting, let's check it out,

```
cd /home/rick; pwd; ls -la
```

![image](https://user-images.githubusercontent.com/67465230/186669320-f7766089-d807-4f88-bddf-1b1e732a67dd.png)

There's a second ingredient. 

Let's read it's content,

```
cd /home/rick; pwd; ls -la; less "second ingredients"
```

![image](https://user-images.githubusercontent.com/67465230/194690291-5598c702-4027-4e2d-82bb-646584a063ba.png)

We got our second ingredient as well.

Now, comes the privilege escalation part. We've to somehow escalate our privileges in order to read the final ingredient which is obviously in /root directory which we can't read without having higher privileges. 

So, we'll check if how many binary we can run without providing root user's password,

```
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/186671129-b279a27e-6553-40de-a0fe-ac07e3fd5bfa.png)

We can run any binary with sudo command in order to escalate our privilege to root user.

So let's establish all content of /root directory,

```
sudo ls /root
```

![image](https://user-images.githubusercontent.com/67465230/186671208-59e5320e-6c00-4179-ae0a-1f8726f099e9.png)

there's our final ingredient. Let's take a look at it,

```
sudo less /root/3rd.txt
```

![image](https://user-images.githubusercontent.com/67465230/194690308-18efbf5b-a96f-44eb-a224-ab3faf9c3a3a.png)

We got our final ingredient.