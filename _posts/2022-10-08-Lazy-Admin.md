---
layout: post
title: "Lazy Admin"
date: 2022-10-08
categories: [Tryhackme, Easy-THM]
tags: [SweetRice, searchsploit, backup, hash, cracking, reverse-shell, burpsuite, perl]
image: ../../assets/img/posts/lazy-admin.png 

---

## Description

Easy linux machine to practice your skills.

|**Room**|[Lazy Admin](https://tryhackme.com/room/lazyadmin)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[MrSeth6797](https://tryhackme.com/p/MrSeth6797)|

---

After deploying machine, we'll start with enumerating it with nmap.

```bash
nmap -sC -sT -sV -O -p- 10.10.127.130
``` 

![image](https://user-images.githubusercontent.com/67465230/186403736-aff6ef5d-3def-4ab3-bf5f-855b70cfab6d.png)

there are 2 open ports found. 

let' visit http://10.10.127.130,

![image](https://user-images.githubusercontent.com/67465230/186403817-ac1abb31-b0b0-4e9e-a937-26b9a3947b50.png)

it's a simple apache2 server webpage. So from here, we'll do directory busting in order to find something interesting,

```bash
gobuster dir -u http://10.10.127.130/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186403872-d06d1d40-2721-463f-9924-9d498f02e307.png)

there's a directory called /content. We'll visit this path,

![image](https://user-images.githubusercontent.com/67465230/186403909-178ee88c-13a9-4408-8f79-81a8df76b94d.png)

It says about **SweetRice**. Never heard of that. Let's search it on **searchsploit**,

```bash
searchsploit sweetrice
``` 

![image](https://user-images.githubusercontent.com/67465230/186403952-31e7abc7-a1bd-4b21-8627-b8d399e5d21c.png)

it has many exploit or vulnerability but there are 2 interesting vulnerability:

 - **Arbitrary File Upload**
 - **Backup Disclosure**

Let's see content of vulnerability,

```bash
searchsploit -x php/webapps/40718.txt
``` 

![image](https://user-images.githubusercontent.com/67465230/186404016-1f7cc1de-c868-4597-a8f2-c59ebbc970d2.png)

there's a /inc directory within url. So let's check if it's there or not

Again using gobuster to do internal directory busting,

```bash
gobuster dir -u http://10.10.127.130/content -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt 2>/dev/null
``` 

![image](https://user-images.githubusercontent.com/67465230/186404082-868549a1-c690-48d2-aa37-eebe022d57eb.png)

this result in many path where we can visit.

visit http://10.10.127.130/content/as/,

![image](https://user-images.githubusercontent.com/67465230/186404135-e437ecfa-db72-42b4-a479-d162fc3e60e2.png)

there's a login page and we've to know the username and password to login to it.

And with internal dir-busting, I know that there's a **/inc** directory, so let's visit http://10.10.127.130/content/inc,

![image](https://user-images.githubusercontent.com/67465230/186404350-ea1c52df-a808-49ac-9f42-af66d1984460.png)

after scrolling down, there's a **mysql_backup** folder present. So I go into that folder,

![image](https://user-images.githubusercontent.com/67465230/186405159-6dd1d41c-2e72-439d-9b16-cfd1bce0d9c7.png)

looks like this contains some information regarding login page. Let's download this.

when I opened this, many things we're found. So I tried to narrow result using grep command,

```bash
cat mysql_bakup_20191129023059-1.5.1.sql | grep passwd
``` 

![image](https://user-images.githubusercontent.com/67465230/194688101-2bcf91f9-272e-45d9-aa90-34a9e919ea11.png)

AAHA!! There a username **manager** and a  hashed password. 

Let's first identify what is this hash using **hash-identifier**,

![image](https://user-images.githubusercontent.com/67465230/194688113-0988a451-dac7-4639-9d0e-91def5c6ce0e.png)

It's **MD5**. Great, now we'll use hashcat to crack this password.

```bash
hashcat -m 0 42f749ade7f9e195bf475f37a44cafcb /usr/share/wordlists/rockyou.txt
``` 

![image](https://user-images.githubusercontent.com/67465230/194688124-0c61c704-f886-46c9-b65f-daae2bd4823b.png)

We now have username and password to login. 

Let's login using **manager:Password123**,

![image](https://user-images.githubusercontent.com/67465230/186405348-61423a4d-1c5c-432a-beaf-ba96ae0dc427.png)

We got in. Now we'll look where we can upload files, if we can.

And there I found where we can upload files,

![image](https://user-images.githubusercontent.com/67465230/186405426-d9ab5908-6e93-4f1b-89da-dc8f2cd3ce64.png)

looks like we can upload our php reverse shell here to get a reverse connection from machine.

When we upload php reverse shell downloaded from https://github.com/pentestmonkey/php-reverse-shell it won't show file which is uploaded. Now edit this downloaded file,

![image](https://user-images.githubusercontent.com/67465230/186405477-c61f5103-1418-441f-8e4e-342a542355ab.png)

change the ip to **tun0 ip** and port to specific one.

When uploading the file, it seems that it's blacklisting **.php** extension, so we'll try to fuzz other php extension using burp.

Intercept the request and send it to intercept,

![image](https://user-images.githubusercontent.com/67465230/186405508-b38db7c4-cb7f-48e5-ba65-416439cd4810.png)

adding mark where we'll fuzz and setting payload,

![image](https://user-images.githubusercontent.com/67465230/186405547-944de549-edcf-46f7-93ce-70104705b137.png)

these are different php extensions that we're going to fuzz on request to check which extension is able to bypass blacklist filters.

we got to know that php5 extension is allowed. So let's upload reverse shell with php5 ext.

![image](https://user-images.githubusercontent.com/67465230/186405579-05b98358-e782-4a13-9c7e-d0fe2e9115a6.png)

the file appears here. 

Now while our netcat listener is listening for connection using `nc -nvlp 1234`, let's click on this file,

![image](https://user-images.githubusercontent.com/67465230/186405729-711f5410-2748-41ed-98e2-34c2d5ef0a12.png)

we got user www-data connection. Let's see if we can access home directory content,

And now, we have an unprivileged shell. In many CTFs, you need to escalate privilege from www-data to a regular user account in order to obtain the user flag, but in this case we can simply **cd /home**,

![image](https://user-images.githubusercontent.com/67465230/186405792-af12e10e-3780-445f-8a1a-451b52646d80.png)

**itguy** was a directory we found inside home directory. Going in there and listing everything,

![image](https://user-images.githubusercontent.com/67465230/186405857-52aa5a40-6074-4a05-b1a9-01c8d31fed32.png)

there's a user flag.

Now, taking look again at all files present in itguy directory, there's a backup.pl file. Let's take a look inside of it,

![image](https://user-images.githubusercontent.com/67465230/186405893-cc3c6cbb-5e1b-4e72-b9b3-be1426bc780d.png)

seems like this is script for system shell. Let's took at a look of copy.sh file content,

![image](https://user-images.githubusercontent.com/67465230/186406273-fcec6dde-f973-4b6c-9e36-a0c6f004fc46.png)

this is reverse shell to elevate privilege. Now we only have to change IP and desired port.

After enumeration I found that this file is writeable, so we'll put content into this file,

![image](https://user-images.githubusercontent.com/67465230/186406515-59503e76-0f72-4f07-a711-7e50bbd7207f.png)

and now checking for binaries which we can run as **sudo**.

![image](https://user-images.githubusercontent.com/67465230/186406569-6ca334e4-7a2f-4475-bcc8-bda5f3986508.png)

we can run **perl** binary as **sudo** to elevate our privilege.

Now that we've echoed the reverse shell in copy.sh file, start netcat listener on new terminal window on 4444 port and  let's run the backup.pl file,

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
``` 

![image](https://user-images.githubusercontent.com/67465230/186406617-db967210-a333-46fc-bdad-06324a075643.png)

as soon as this netcat got connection, we gain elevated shell, meaning system shell,
![image](https://user-images.githubusercontent.com/67465230/186406675-6c9e7699-9e10-42a6-8fbe-ee8551d1d844.png)