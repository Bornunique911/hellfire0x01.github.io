---
layout: post
title: "Year Of The Rabbit"
date: 2022-10-09
categories: [Tryhackme, Easy-THM]
tags: [rustscan, dirsearch, disabling-javascript, ftp, hydra, brainfuck, CVE-2019-14287]
image: ../../assets/img/posts/yearoftherabbit.png 

---

## Description

Time to enter the warren...

|**Room**|[Year Of The Rabbit](https://tryhackme.com/room/yearoftherabbit)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[MuirlandOracle](https://tryhackme.com/p/MuirlandOracle)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.161.34
```

![image](https://user-images.githubusercontent.com/67465230/187016372-53371b25-a982-4350-a309-666ef527c3c9.png)

We got 3 open ports. Lets scan then them using nmap,

```bash
nmap -sC -sV -p21,22,80 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187016387-ec69f638-36d9-457b-ad12-42d0de60ef1a.png)

Looks like port 21 is running ftp service, port 22 is running ssh and port 80 is running apache server. Let's start with enumerating port 80 first.

Visit http://10.10.161.34,

![image](https://user-images.githubusercontent.com/67465230/187016399-eede30fe-09f0-475f-975e-5a2a0854432b.png)

I landed on default page of apache2 debian. Nothing interesting.

Now quickly find hidden directories,

```bash
dirsearch -u http://10.10.161.34 -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187016410-ebae7fed-67a5-402c-b07b-0489a4d65b5d.png)

after the directory busting is completed, I got /assets as a hidden directory.

Quickly navigate to http://10.10.161.34/assets,

![image](https://user-images.githubusercontent.com/67465230/187016419-372fbaf4-e1d3-4f26-afd4-f4c6bfc0f512.png)

there are 2 files in which 1st is a video of RickRolled.mp4 and 2nd is a style.css file.

Checking the css file,

![image](https://user-images.githubusercontent.com/67465230/187016423-58548591-f3da-4d15-a017-88cade527ed3.png)

there is a **/super_s3cr3t_fl4g.php** page which is hidden in CSS comments.

I decided to visit to this hidden page http://10.10.161.34/super_s3cr3t_fl4g.php,

![image](https://user-images.githubusercontent.com/67465230/187016429-33142434-f0ca-4551-bb09-032b393d800d.png)

I stumbled with this alert box saying, "Word of advice... Turn off your javascript". 

I didn't knew how to do it so for the time being, I clicked on **OK** button and I was redirected to Rick Astley video and got *Rick Rolled*,

![image](https://user-images.githubusercontent.com/67465230/187016443-37d25124-1132-490f-965c-61246e333f5b.png)

Now, I went back to the same prompt and did some research which means that in order to load the webpage correctly, we gotta turn off the javascript from browser (which is enabled by-default).

Now, we can disable javascript in firefox by navigating to `about:config` then search for "javascript". It will show that `javascript.enabled`, change the value to 'false',

![image](https://user-images.githubusercontent.com/67465230/187016451-f0476f22-b663-42fa-ab67-f627ed7439f2.png)

Now, load this page again,

![image](https://user-images.githubusercontent.com/67465230/187016462-a62bdaa1-cb2e-49be-b695-77b77d95b66d.png)

I got a screen with a message with a hint on it, *The hint is in the video*.

I played this video and at 56 seconds I found the hint in the audio:

>"I'll put you out of your misery **burp** you’re looking in the wrong place"

Firing up the burpsuite and let it intercept the request and click “Forward” and see what comes next, 

![image](https://user-images.githubusercontent.com/67465230/187016473-c8f2ac92-ea99-43c5-a07e-79f5c094c778.png)

and looking carefully at the request reveals that there is a page at which I stopped, **/intermediary.php** with a parameter of **hidden_directory**. Looks good.

Navigate to http://10.10.161.34/WExYY2Cv-qU,

![image](https://user-images.githubusercontent.com/67465230/187016486-397d5d5b-33ea-4965-b598-18093ebf1795.png)

There is an image on the directory. Downloading this image for further inspection.

Since this is an image file, so I tried to extract the hidden data (with binwalk, steghide and what not) but didn't get anything. Alas, I ran strings command on it,

```bash
strings Hot_Babe.png
```

![image](https://user-images.githubusercontent.com/67465230/194744257-f96957c2-449d-4b3e-9992-af93a45e4945.png)

scrolling till last, I got the username **ftpuser** and a bunch of password for ftp service.

Copy these passwords and paste into ftppass file and then using hydra to identify correct password of ftp service by brute forcing,

```bash
hydra -l ftpuser -P ftppass ftp://10.10.163.42:21 -f
```

![image](https://user-images.githubusercontent.com/67465230/194744281-8e2354d6-9f65-4e73-a29f-b3c4815ba9fa.png)

we got the correct password.

Now, that I got the credentials for ftp service, I can now access the service,

```bash
ftp 10.10.163.42
```

![image](https://user-images.githubusercontent.com/67465230/187016516-ab194de7-6751-4aea-a637-f014aaa7fd58.png)

and I get in.

Now, enumerating directory using `ls -la`,

![image](https://user-images.githubusercontent.com/67465230/187016524-10ca6289-fa48-48c5-901e-f6f3e6a926f8.png)

Hmm, there is a text file named **Eli's_Creds.txt**. 

I can download this file on my kali machine using,

```bash
get "Eli's_Creds.txt"
```

Now reading the content of the file we downloaded,

![image](https://user-images.githubusercontent.com/67465230/187016870-4f2e28be-bc4a-4abd-93f6-bf504616cd8c.png)

Seems like this is **Brainfuck** language.

I google about this and got this amazing [BrainFuck Decoder](https://www.dcode.fr/brainfuck-language),

![image](https://user-images.githubusercontent.com/67465230/194744318-5ddbf5f4-de2d-4135-9f75-28f44e5509fe.png)

after decoding the script, I got the username and password (maybe for SSH service!). 

I can now dive into system via ssh,

```bash
ssh eli@10.10.163.42
```

![image](https://user-images.githubusercontent.com/67465230/187016931-4828c8d2-828f-480b-bdf8-a647878c4323.png)

I got in.

Now Enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187016944-5451d79f-505c-47ef-b0c0-2a1db0a948c7.png)

I got many other directories. 

Navigating back one directory and there is another user, **gwendoline**

![image](https://user-images.githubusercontent.com/67465230/187016954-ef798df4-d34c-42be-a7ab-c8973d28c817.png)

Now, I tried to work on the only clue, the message from Root, "leet s3cr3t hiding place", that Gwendoline user supposedly knows about, find out

```bash
find / -name s3cr3t -type d 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187016976-c8a7f04d-4d64-4802-8eca-19d0111a9614.png)

the s3cr3t file can be find in **/usr/games/s3cr3t** directory.

Enumerating directory,

```bash
ls -la /usr/games/s3cr3t
```

![image](https://user-images.githubusercontent.com/67465230/187016985-00f89300-9a06-4535-b250-8c6aa9c606e5.png)

the message file is hidden.

Reading the content of the file,

![image](https://user-images.githubusercontent.com/67465230/187017008-0ec54572-0d3d-40e4-b896-64bcccd847ce.png)

There is a password mentioned in the message file of Gwendoline user. Sweet.

Switching user to Gwendoline,

```bash
su gwendoline
```

![image](https://user-images.githubusercontent.com/67465230/187017027-b56f952e-8626-45b7-8c3f-daeb307d2880.png)

Now, enumerating home directory of Gwendoline user, I found the user.txt file,

![image](https://user-images.githubusercontent.com/67465230/187017032-6eb4eaf9-4c11-4d94-addc-520a775ab35a.png)

Great. Moving forward to privilege escalation part, I tried to list all those binaries which I can run as sudo to escalate privileges without providing password,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/187017047-bfb72527-a51c-4a04-babf-e09a388cc833.png)

it shows that **/usr/bin/vi** editor can be run on **/home/gwendoline/user.txt** file, when run, can escalate the user privileges.

Now, trying to escalate privilege,

```bash
sudo /usr/bin/vi /home/gwendoline/user.txt
```

![image](https://user-images.githubusercontent.com/67465230/187017084-d0e81d1e-e17b-49b5-bbc3-c4be770181c2.png)

User Gwendoline can't run /usr/bin/vi on stated file as sudo user because they are not part of sudoers file. Now comes the CVE-2019-14287.

>**CVE-2019-14287**: Joe Vennix found that if you specify a UID of -1 (or its unsigned equivalent: 4294967295), Sudo would incorrectly read this as being 0 (i.e. root). This means that by specifying a UID of -1 or 4294967295, you can execute a command as root, despite being explicitly prevented from doing so. 

I take advantage of this flaw to escalate the privilege to root user,

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

![image](https://user-images.githubusercontent.com/67465230/187017098-aa2b34bb-164b-498c-a616-8b97c8e2ae4e.png)

and the vi editor opens up. I tried to escape the shell using `!sh`.

I became root user,

![image](https://user-images.githubusercontent.com/67465230/187017107-45ac2cfa-4b37-4bcc-8cb9-5688e79d938f.png)

and now I got the root flag by navigating to root directory.