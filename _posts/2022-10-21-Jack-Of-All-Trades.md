---
layout: post
title: "Jack-Of-All-Trades"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [rustscan, base64, cyberchef, hex, ROT13, steghide, Command-Injection, hydra, bruteforcing, scp, SUID, strings]
image: ../../assets/img/posts/jackofalltrades.png 

---

## Description

Boot-to-root originally designed for Securi-Tay 2020

|**Room**|[Jack-Of-All-Trades](https://tryhackme.com/room/jackofalltrades)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[MuirlandOracle](https://tryhackme.com/p/MuirlandOracle)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.172.45 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/186387137-17ae046a-8337-43a2-9531-ebf8945f4f38.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sC -sV -p22,80 10.10.172.45 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/186387209-e6152e94-0d2b-483d-99d7-68cb32cb0216.png)

Result scan shows that port 22 is running apache webserver and port 80 is running ssh service? That strange.

While visiting http://10.10.172.45:22, browser shows that the webpage is restricted,

![image](https://user-images.githubusercontent.com/67465230/186387282-8e6835bb-cf99-473a-86f3-dc3149dd4696.png)

We can bypass this restriction in firefox in following manner: navigate to `about:config` and we will get a message that we're voiding our warranty (for free software). Click on agree and we'll be shown a list of configurations. From there, search for `network.security.ports.banned.override`. In some versions of Firefox this might show nothing (in which case right-click anywhere on the page, choose `new -> String` and use the search query as the preference name) 

![image](https://user-images.githubusercontent.com/67465230/186389571-dd4158f4-c27c-49c8-84f4-5daf46ef3c37.png)

Whether you’ve had to create a new entry or not, add or change the “Value” field to be `22`. 

We can now go back to webpage and reload and it will load properly,

![image](https://user-images.githubusercontent.com/67465230/186387352-37ef28e1-d154-487c-9cea-d3ac63b88a57.png)

we are welcomed to a webpage. Enumerating it and we won't get anything. So I decided to look at source code page,

![image](https://user-images.githubusercontent.com/67465230/186387413-04d34951-332e-486e-bb5a-d964091cf36f.png)

there's a path to recovery.php page and a base64 encoded string.

First, let's decode the string,

```bash
echo "base64-string" | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/186387461-2c3ae5c0-7650-4b1f-99b7-80eb4391c310.png)

we got the decoded message and there is a name mentioned, *Johny Graves*. 

I decided to google this name and there appears a 1st link of the page, 

![image](https://user-images.githubusercontent.com/67465230/186387514-ce3327b6-4263-430f-ade3-2ba4e8ea3f15.png)

Navigating on it and we can see the message left by *Johny Graves* that his favorite crypto method and is that first encode message with a ROT13 cipher then convert it with hex and then convert it into Base32,

![image](https://user-images.githubusercontent.com/67465230/186387597-5a6006e9-ec93-4bf3-a8cf-79cc6456d19c.png)

Now let's visit http://10.10.172.45:22/recovery.php and we will land on a login form page, 

![image](https://user-images.githubusercontent.com/67465230/186387938-334bb12e-12a9-413a-82e6-cbbf102e40f6.png)

we can't login since we don't have credentials. 

So I decided to look around and I found something interesting on source page,

![image](https://user-images.githubusercontent.com/67465230/186387986-51e92f79-b897-4b8d-8294-530ae56c53a4.png)

it is an encoded string. 

Let's visit [Cyberchef](https://gchq.github.io/CyberChef) and from there we can use base32 format decoding,

![image](https://user-images.githubusercontent.com/67465230/186388043-88d90065-9e75-4825-bebd-69c5cbca5271.png)

this is definitely hex format!

Let's convert this string again,

![image](https://user-images.githubusercontent.com/67465230/186388083-7e7bd3a9-711b-425c-91a4-8d9681b1be59.png)

Output looks good. Using ROT13 and we'll get our answer,

![image](https://user-images.githubusercontent.com/67465230/186388119-d5376f21-668f-4a1d-b4cf-d6a3b6fee9d8.png)

Looking at the hint that jack left us, when navigating to the link, we got redirected to wikipedia of Stegosauria,

![image](https://user-images.githubusercontent.com/67465230/186388163-1406d29d-cf32-497b-be7b-dcb74c1400af.png)

There is a Stegosaurus sitting there on the homepage,

![image](https://user-images.githubusercontent.com/67465230/186388245-3e211752-a639-4a8b-95d0-df75308aacaf.png)

let's download this image and try to extract data with and password we found earlier, 

```bash
steghide extract -sf stego.jpg
```

![image](https://user-images.githubusercontent.com/67465230/186388281-e553efa4-3bf9-49db-8e9c-6b89a6ad1ad0.png)

there is a creds.txt file which got extracted.

Reading creds.txt file,

![image](https://user-images.githubusercontent.com/67465230/186388335-016ea376-4f49-43e1-a684-b587b6425114.png)

we are almost there but we didn't get the creds. 

Now, we can try to fuzz directories using gobuster,

```bash
gobuster dir -u http://10.10.249.96:22/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -q 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186388388-5eb5736b-5b9d-4e9d-8a9b-c907e398f44a.png)

there is an asset directory.

Navigating to http://10.10.249.96:22/assets which contains some images so we will download them,

![image](https://user-images.githubusercontent.com/67465230/186388455-768a524a-c689-474d-aecf-c814adeb4409.png)

Now, using steghide on header.jpg image file,

```bash
steghide extract -sf header.jpg
```

![image](https://user-images.githubusercontent.com/67465230/186388543-771efb7d-93cd-4c8f-90a6-4b3fec932bfb.png)
there is a file that got extracted named cms.creds.

Reading the content of cms.creds file, 

![image](https://user-images.githubusercontent.com/67465230/186388612-cec64c90-3138-432c-8772-7d4e4808de7d.png)

we got the credentials.

Using these creds, I tried to login and got in! Now, I ended up on this page where there is a clear message for us that we need to provide ***cmd*** parameter in GET request and application will run it (command injection),

![image](https://user-images.githubusercontent.com/67465230/186388665-85badcdb-1b92-4fd2-8621-32df48b6afda.png)

Now,let's provide a parameter, *cmd* and a linux command, `id`,

```bash
?cmd=id
```

![image](https://user-images.githubusercontent.com/67465230/186388749-6cce4d55-dd0b-4dc3-9dc7-83afb13f782f.png)

this will return the id of the current user.

Now, I will try to enumerate home directory and see what users are there,

```bash
?cmd=ls -la /home/
```

![image](https://user-images.githubusercontent.com/67465230/186388814-ac8954cb-5edd-42e0-8473-e787d80214ba.png)

there is one directory named jack and a file named jacks_password_list and this file seems very juicy.

Reading the content of this file,

```bash
?cmd=cat /home/jacks_password_list
```

![image](https://user-images.githubusercontent.com/67465230/186388886-54c3d941-a85b-4937-870a-b55061c5097b.png)

we got a possible password list.

We can use hydra tool to brute-force username and password,

```bash
hydra -l jack -P jack_creds ssh://10.10.249.96:80
```

![image](https://user-images.githubusercontent.com/67465230/186388956-a61a9baf-e7dd-4d59-896c-4e764bbc69f6.png)

we got a password for the corresponding user.

Let's try to hop into system via ssh,

```bash
ssh jack@10.10.249.96 -p 80
```

![image](https://user-images.githubusercontent.com/67465230/186388997-aab1e7ac-775c-4445-952c-942bb96c6a52.png)

we got in!! Enumerate directory and we will get our user image? 

Downloading this image on our local system using `scp` command,

```bash
scp -P 80 jack@10.10.249.96:/home/jack/user.jpg .
```

![image](https://user-images.githubusercontent.com/67465230/186389045-d960f0d5-283e-4bdd-b876-ffe214580393.png)

Open image and we will get our user flag,

![image](https://user-images.githubusercontent.com/67465230/186389133-80220d4c-a049-4de4-aebe-2208805a182f.png)

Now comes the privilege escalation part where we will list all the binaries which we can run using `sudo`,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/186389183-a6ce3d10-3f75-4c13-9f6e-606244142bfb.png)

we can't run any binary as sudo while we are jack user.

We can try to find those binaries which has SUID bit set on them, 

```bash
find / -perm -4000 -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186389256-f97875f7-9867-4d40-9336-526bfb8fcd70.png)

there is a binary */usr/bin/strings* which can run to elevate our privileges.

We can search for [strings | GTFOBins](https://gtfobins.github.io/gtfobins/strings/#suid) method on how to perform privilege escalation as we can try to read root flag using strings binary,

```bash
/usr/bin/strings /root/root.txt
```

![image](https://user-images.githubusercontent.com/67465230/186389386-51a7da66-0339-402d-a9a6-c4f5f11e3573.png)
