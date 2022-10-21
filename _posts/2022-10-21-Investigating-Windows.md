---
layout: post
title: "Investigating Windows"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [Windows, Evenevent-logs]
image: ../../assets/img/posts/investigatingwindows.png 

---

## Description

A windows machine has been hacked, its your job to go investigate this windows machine and find clues to what the hacker might have done.

|**Room**|[Investigating Windows](https://tryhackme.com/room/investigatingwindows)|
|:---:|:---:|
|**OS**|Windows|
|**Difficulty**|Easy|
|**Creator**|[tryhackme](https://tryhackme.com/p/tryhackme)|

---

Q. Whats the version and year of the windows machine?

For this, I opened “System Information” with the “Server Manager”,

![image](https://user-images.githubusercontent.com/67465230/197191767-2fa0a36c-1b2b-4f9b-9a34-de08e026fa2e.png)

Q. Which user logged in last?

To list all users I entered,

```cmd
net users
```

![image](https://user-images.githubusercontent.com/67465230/185960075-11fa724d-2330-44bc-a311-ef0b1380997c.png)

Because I just logged in, I entered the “Administrator” user to inspect the logon time and to get some additional informations.

Q. When did John log onto the system last? Answer format: MM/DD/YYYY H:MM:SS AM/PM

```cmd
net user ****
```

![image](https://user-images.githubusercontent.com/67465230/185961903-c9296375-b438-467a-ab37-e7879fd6f76f.png)

Q. What IP does the system connect to when it first starts?

When machine was booting up, we are shown the IP address and this was the IP address that machine first connects to when get's booted up,

![image](https://user-images.githubusercontent.com/67465230/197192009-2c7fe433-a79a-4d99-b9e1-b82ee02bcb6a.png)

Q. What two accounts had administrative privileges (other than the Administrator user)? Answer format: username1, username2

We can take a look at localgroup of administrator and check the members,

```cmd
net localgroup administrators
```

![image](https://user-images.githubusercontent.com/67465230/185963227-99a0717d-3ca3-43f4-a950-c69f361536b8.png)

Q. Whats the name of the scheduled task that is malicious?

Open up the "Task Scheduler" and found the malicious scheduled task,

![image](https://user-images.githubusercontent.com/67465230/197192250-8978e240-2f59-4a6c-bf5c-f36d847a21e5.png)

Q. What file was the task trying to run daily?

With the task above under “Actions” I got this answer,

![image](https://user-images.githubusercontent.com/67465230/197192455-7b4129b8-b576-43ad-85a2-97783977591b.png)

Q. What port did this file listen locally for?

A. Here the “-l” flag got set to listen for the entered port.

Q. When did Jenny last logon?

We can take a look at Jenny user,

```cmd
net user Jenny
```

![image](https://user-images.githubusercontent.com/67465230/185964519-0a0c99e4-9202-4266-a0fc-1577e6d5c8b9.png)

Q. At what date did the compromise take place? Answer format: MM/DD/YY

While exploring the system some more, I found a suspicious directory called "/TMP" on the "C:\" location,

![image](https://user-images.githubusercontent.com/67465230/197193100-0d402171-4f7a-4c62-a249-d22baa2884b8.png)

Q. At what time did Windows first assign special privileges to a new logon? Answer format: MM/DD/YYYY HH:MM:SS AM/PM

To get the answer to this question I opened the “Event Viewer” and inspected the “Security” logs. For this I have oriented myself according to the date of the compromise,

![image](https://user-images.githubusercontent.com/67465230/185965893-39b1101d-2235-49d9-b643-318ee5c7a19d.png)

Q. What tool was used to get Windows passwords?

In the “/TMP” directory were some more files and one well-known tool with an output file,

![image](https://user-images.githubusercontent.com/67465230/197194093-a7ff21f5-fae8-4bef-ade3-c9e67a27ab2d.png)

Q. What was the attackers external control and command servers IP?

I got the answer for this question when I solved the last question of the room looking at the “hosts” file, 

![image](https://user-images.githubusercontent.com/67465230/185966702-1262a3e5-0fe5-4ada-9525-4b5ed399bfbb.png)

Q. What was the extension name of the shell uploaded via the servers website?

The website of the server was located at "C:\intetpub\wwwroot" and here I got the files,

![image](https://user-images.githubusercontent.com/67465230/197193893-b00274fc-d958-4b1c-9ee9-69558ab706a2.png)

Q. What was the last port the attacker opened?

For this I had to take the hint and went to the Firewall options. The first Inbound Rule was to allow outside connections on the local port 1337,

![image](https://user-images.githubusercontent.com/67465230/197193735-7d0b6fdd-171f-4c94-8604-6dff4fb36b6c.png)

Q. Check for DNS poisoning, what site was targeted?

To get to the DNS record like Linux, I searched for the hosts file on Windows. I got the location of it from [this site](https://www.thewindowsclub.com/hosts-file-in-windows). In here I got the hostname and the IP address from the attacker for question 13,

![image](https://user-images.githubusercontent.com/67465230/197193647-877b8bb3-3f77-4918-a071-803e28f1cd0f.png)
