---
layout: post
title: "Brooklyn Nine Nine"
date: 2022-03-23
categories: [Tryhackme, Easy-THM]
tags: [security, nmap, gobuster, pentest, ftp, less, nano, steganography]
image: ../../assets/img/posts/BrooklynNineNine.png 

---

## Description

This room is aimed for beginner level hackers but anyone can try to hack this box. There are two main intended ways to root the box. 

|**Room**|[Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Fsociety2006](https://tryhackme.com/p/Fsociety2006)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.137.33 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/159624485-85546757-116b-4b2d-a382-8f30c227dd6f.png)

we got 3 open ports. Let's scan them in detail with nmap.

```bash
nmap -sV -sC -p21,22,80 10.10.137.33 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/159624521-6dc8a11c-4905-4d46-ac04-2d4d3839f66e.png)

Scan results describes that port 21 is running ftp service with anonymous login, port 22 is running ssh service and port 80 is running apache webserver. Lets enumerate ftp service.

Access the ftp service,

```bash
ftp 10.10.137.33
```

![image](https://user-images.githubusercontent.com/67465230/159624533-138e04ad-8e80-4da5-b6c9-cf72525bf9e9.png)

Enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/159624544-fb19dd61-00b9-40f1-94e2-bf9a11ce6ba7.png)

we got a note file. 

Lets download this file onto our system,

```bash
get note_to_jake.txt
```

![image](https://user-images.githubusercontent.com/67465230/159624559-fd3fd870-d0e4-404d-aca6-8386f8329285.png)

Read the content of this file,

![image](https://user-images.githubusercontent.com/67465230/159624572-1833542f-2ba5-417f-ba64-b66644218aa2.png)

seems like we can get a hold of password of holt user.

Now, time to enumerate port 80. Visit http://10.10.137.33,

![image](https://user-images.githubusercontent.com/67465230/159624584-5dd78bcb-ab0f-4f7f-b966-c4b0b31e4e36.png)

we got a webpage with image of BROOKLYN NINE-NINE.

While reading the source code of this webpage,

![image](https://user-images.githubusercontent.com/67465230/159624589-bd051787-ea0a-4a6a-9f67-a253bdb1a977.png)

I realised that there must be some kind of data which is hidden in image. So we can use the technique, Steganography to extract the hidden data from an image.

So lets start by download the image from the following page,

![image](https://user-images.githubusercontent.com/67465230/159624598-875543a2-fb26-4ab5-8356-28503e33023d.png)

I tried steghide, binwalk, zsteg to extract the information but they are unable to extract the information. So I used a tool called stegseek, using which tool can automatically crack the password (by using provided wordlist) and extract the file content,

```bash
stegseek --crack brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/159624611-9a60e929-9bca-4b55-aff4-3e138097134c.png)

Taking a look at extracted file,

![image](https://user-images.githubusercontent.com/67465230/159624625-1bf01888-daf4-4506-899e-e50a00c3538a.png)

we got the holt user password.

Now, we can get into machine via ssh service,

```bash
ssh holt@10.10.137.33
```

![image](https://user-images.githubusercontent.com/67465230/159624632-c2323f02-97eb-463a-93ae-5f8151b62be6.png)

Enumerate directory and we found user flag,

![image](https://user-images.githubusercontent.com/67465230/159624643-dfb67411-9d5d-4ee6-bddd-cf0fe653468a.png)

Now, time for privilege escalation. We scan list all binaries which can be run as sudo,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/159624664-fe8f64c0-20a5-40f4-8e49-34c1b2db8b06.png)

**/bin/nano** can be run as sudo.

We can look for gtfobins to escalate our privileges,

![image](https://user-images.githubusercontent.com/67465230/159624672-dfc60c9f-fffc-4d6f-b274-ae0e11dc08b6.png)

But, using nano binary, we can read the root.txt file,

```bash
sudo /bin/nano /root/root.txt
```

![image](https://user-images.githubusercontent.com/67465230/159624686-3158e41d-f947-4460-b75b-f34d71a83493.png)

Now, there is another intended way, as per box's description. So I tried to look for binaries which have SUID bit set on them,

```bash
find / -perm -04000 -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/159624698-dc2689a5-5be1-4b4c-b7ad-d715ad9e1ac8.png)

**/bin/less** binary has SUID bit set on it.

So, I know a place where we can find the escalation technique, gtfobins,

![image](https://user-images.githubusercontent.com/67465230/159624707-5496c14a-de44-4b45-9b22-0ede18fd0be8.png)

we can use this command to read the root.txt file,

```bash
/bin/less /root/root.txt
```

![image](https://user-images.githubusercontent.com/67465230/159624718-deeff988-bcba-4089-810a-31ac03c32be1.png)
