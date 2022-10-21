---
layout: post
title: "ColddBox Easy"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, dirsearch, wordpress, wpscan, reverse-shell, find, SUID, GTFObins]
image: ../../assets/img/posts/colddboxeasy.png 

---

## Description

An easy level machine with multiple ways to escalate privileges.

|**Room**|[ColddBox Easy](https://tryhackme.com/room/colddboxeasy)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Coldd](https://tryhackme.com/p/Coldd)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.241.140 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187383079-71d24309-2ea9-4063-82e3-96d8a5c94562.png)

we got 2 open ports. Lets scan these ports with nmap,

```bash
nmap -sC -sV -p80,4512 10.10.241.140 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187383272-d09591ad-31fa-4852-84cd-5b9106e0d95c.png)

looks like port 80 is running apache2 webserver and port 4512 is running ssh service. Let's enumerate port 80 first.

Visit http://10.10.241.140,

![image](https://user-images.githubusercontent.com/67465230/187383338-2ae6d997-4c79-4cda-80c5-5a2b524791ba.png)

we got a wordpress website. We can further enumerate this machine but there is nothing to bother about much.

Now, let's find hidden directories using dirsearch,

```bash
dirsearch -u http://10.10.241.140 -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187383385-9e8671a7-c31d-4bb3-8152-030e319d64ea.png)

we got some hidden directories. Let's explore the /hidden path first.

Visit http://10.10.241.140/hidden/,

![image](https://user-images.githubusercontent.com/67465230/187383439-974153d3-270a-46a6-b2e9-2031bf7217f2.png)

we got a message that c0ldd user has changed Hugo user password, means there is a c0ldd, hugo user (username enumeration).

Now, we can use wpscan tool to gather some information about the website (which is running on wordpress),

```bash
wpscan --url http://10.10.241.140/ -e
```

![image](https://user-images.githubusercontent.com/67465230/187383482-c57023bc-5149-49b3-8243-7ae274ab41d1.png)
![image](https://user-images.githubusercontent.com/67465230/187383530-48fb08d4-a881-46bb-82b8-3cf8c8d5a9c8.png)

-e flag will enumerate the website like WP themes or identification of users.

Now, we know that wpscan has found 3 users, we can now brute force the password by providing the wordlist,

```bash
wpscan --url http://10.10.241.140/ -U hugo,c0ldd,philip -P /usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/187383573-18e7584e-e36a-42ba-8ff7-51d9060a7cf2.png)

we now have the c0ldd user password. Great. 

Navigating to http://10.10.241.140/wp-login.php,

![image](https://user-images.githubusercontent.com/67465230/187383619-adb02ba4-6524-4bfd-976f-e4580595fc15.png)

we got a login page. Let's use the credentials we found earlier,

And we landed on the dashboard of the website,

![image](https://user-images.githubusercontent.com/67465230/187383697-50bc00fd-ba13-40ef-a836-f36fc4e31a3d.png)

After some enumeration, there is a Themes section in Appearance menu. I was interested to look inside of it.

So I decided to see what is there in it and there is file named 404.php which seems interesting to me. Maybe I can put a php reverse shell there and connect to the machine? Let's find out.

![image](https://user-images.githubusercontent.com/67465230/187383742-3f28b6de-42e0-42a6-8285-1d348bd5ee51.png)

Since this is a php extension file, I put a reverse-shell there with changing my IP and port.

Now, before triggering the shell, lets first start the netcat listener using `nc -nvlp 4444` and then we can navigate to http://10.10.241.140/wp-content/themes/twentyfifteen/404.php, as we know that theme used in this website is twentyfifteen (does makes sense!).

And we got a shell,

![image](https://user-images.githubusercontent.com/67465230/187383809-cecb23ca-477f-4195-a744-b8966eab5b0d.png)

Now since we have a non-functional shell and we can't access the tty with no job controls, we can upgrade the shell using following commands:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
ctrl + Z
stty raw -echo; fg
stty rows 38 columns 116
```

Enumerating home directory reveals that there is a c0ldd user directory. Navigating into it and enumerating is reveals that there is a user flag,

![image](https://user-images.githubusercontent.com/67465230/187383852-a5b373b1-dedb-4fef-836e-56e22e47af9c.png)

Reading the content of user flag throws us an error of permission denied,

![image](https://user-images.githubusercontent.com/67465230/187383903-dfb0fce5-31fe-4c48-8be5-c2536c20e206.png)

Looks like we have to elevate our privilege in order to read the flag.

Looking for binaries which I can run as sudo without providing password,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/187383935-8e5cd167-17b1-48ad-a1fd-9193429cfbf1.png)

Since we don't have password for www-data user, we can't run it.

Now, finding those files which has SUID bit set on them,

```bash
find / -perm -04000 -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187383991-3df0208e-8d0e-4d41-ba2f-35addb515add.png)

there is a binary **/usr/bin/find** which has SUID bit set on it, means we can run it to elevate our privileges to root user.

Looking at the method on gtfobins on how can we elevate our privileges, I saw this command,

![image](https://user-images.githubusercontent.com/67465230/187384038-efb8a800-e2ea-49a5-9745-43ad918dc91a.png)

Now, running this command will get us root shell,

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

![image](https://user-images.githubusercontent.com/67465230/187384102-3951a2fd-7f1c-4d4c-8a05-585da356024e.png)

So, we are root. 