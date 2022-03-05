---
layout: post
title: "Kioptrix Level 3"
date: 2022-03-05
categories: [VulnHub, Easy-Vulnhub]
tags: [boot2root]
image: ../../assets/img/posts/Kioptrix-level-3.png

---

## Description

This Kioptrix VM Image are easy challenges. The object of the game is to acquire root access via any means possible (except actually hacking the VM server or player). The purpose of these games are to learn the basic tools and techniques in vulnerability assessment and exploitation. There are more ways then one to successfully complete the challenges. 

|**Box**|[Kioptrix Level 3](https://www.vulnhub.com/entry/kioptrix-level-12-3,24/)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Kioptrix](https://www.vulnhub.com/author/kioptrix,8/)|

---

We'll start with first discovering the IP address of the box using **netdiscover** tool,

```bash
sudo netdiscover -r 10.0.2.0/24
```

![image](https://user-images.githubusercontent.com/67465230/156873616-b359f212-d68d-4676-b0c6-e4febc51afc8.png)

Okay, so the IP address of the box is **10.0.2.68**.

Now, let's start port scanning with **nmap**,

```bash
sudo nmap -A -T4 -p- -oN nmap_scan 10.0.2.68
```

![image](https://user-images.githubusercontent.com/67465230/156873944-f38c00d1-9c69-4153-9304-e70d668d279f.png)

with results we can only see port 22(ssh) and 80(http) open. Let's enumerate port 80.

Let's identify which technologies are running on website,

```bash
whatweb 10.0.2.68
```

![image](https://user-images.githubusercontent.com/67465230/156873950-f9671b72-07f9-4626-a669-83db6a1c14e2.png)

Let's run nikto tool to find if there's something interesting we can find on machine,
 
```bash
nikto --host http://10.0.2.68 --output nikto.html
```

![image](https://user-images.githubusercontent.com/67465230/156873958-9e5a9995-eb5b-4b9f-9859-fc753c7917dd.png)

there's a **phpmyadmin** directory as well. Seems interesting to me.

Now, let us enumerate website by visiting http://10.0.2.68/,

![image](https://user-images.githubusercontent.com/67465230/156873962-e3d59851-c6ed-47bd-baf4-1d461ee3ed52.png)

"Ligoat Security" says this website. Going to **Blog** section,

![image](https://user-images.githubusercontent.com/67465230/156873968-6b0c73b1-4162-4625-a7dc-57846f96ed17.png)

There are 2 main things that I found, first link to gallery directory http://kioptrix3.com/gallery and second username **loneferret**.

Let's visit login section and see if we can login,

![image](https://user-images.githubusercontent.com/67465230/156874010-28dc1e77-2a02-4a11-b9a5-5338c477f7aa.png)

we can't login into this website. But this login page shows that it's made of **LotusCMS**. So I tried to search this on **searchsploit** to see if there's actually any exploit,

```bash
searchsploit lotus cms
```

![image](https://user-images.githubusercontent.com/67465230/156873971-462c7b23-8c4e-4e0e-85ed-44ee5d5d9239.png)

AHH!! there is a metasploit 'rb' module with RCE is possible. Let's type `msfconsole -q` in terminal to fireup metasploit-framework with **-q**(**quiet**) flag to not to print the banner

we'll search for lotus cms,

```bash
search lotus cms
```

![image](https://user-images.githubusercontent.com/67465230/156874031-b021fe61-bbaf-4a0d-8343-49ab31634a8a.png)

we'll be shown with this ruby module. Let's use this module,

setting the options,
 - **set rhosts 10.0.2.68**
 - **set uri /**
 - **options**

![image](https://user-images.githubusercontent.com/67465230/156874032-faf7274f-05ce-4c6e-9030-317f60e13674.png)

after setting everything needed, check for options again for if we missed anything and now we'll run this exploit,

```bash
run
```

![image](https://user-images.githubusercontent.com/67465230/156874035-419ecae7-0d2c-4981-9c6f-e007752b7df6.png)

as soon as this exploit runs, we get a shell and we can confirm it using `whoami` command.

Let's improve our shell's functionality,

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

![image](https://user-images.githubusercontent.com/67465230/156874042-29f30b08-83b9-4daa-8084-b52bdca5b9f6.png)

Viewing some info about user, `whoami` and `id` can do this job well,

![image](https://user-images.githubusercontent.com/67465230/156874045-d0bb45ab-919a-421a-9967-dd485f1e5a67.png)

Let's view /etc/passwd file,

```bash
cat /etc/passwd
```

![image](https://user-images.githubusercontent.com/67465230/156874048-66c00d99-04d8-44c9-a38c-3cc555d06434.png)

we can see that there are 2 users, **loneferret** and **derg**.

navigate to /home dir using `cd` command and establish contents using `ls -la` command,

![image](https://user-images.githubusercontent.com/67465230/156874050-4589601e-6978-41b6-9ea2-830b7281d5d3.png)

user loneferret directory seems interesting. Let's navigate into it,

establishing contents using `ls -la`,

![image](https://user-images.githubusercontent.com/67465230/156874054-c9038346-6de1-4acc-8d5c-07992053e89b.png)

seems like **CompanyPolicy.README** file is interesting. Let's view what is inside of it,

```bash
cat CompanyPolicy.README
```

![image](https://user-images.githubusercontent.com/67465230/156874058-0918ea08-17a8-4e27-a8bd-bc9ffe3eb85e.png)

This policy wants us to use `sudo ht` command to use our newly installed software for editing, creating and viewing files.  Let's do the same,

```bash
sudo ht
```

![image](https://user-images.githubusercontent.com/67465230/156874064-7c3867e6-f804-434f-b3cc-e9403f377351.png)

since we have no passwords means we can't be able to use this command.

now navigating to **/home/www/kioptrix3.com** directory and establishing contents using `ls -la`,

![image](https://user-images.githubusercontent.com/67465230/156874067-11920a78-52e2-4b50-a273-3b78361cef36.png)

there are many directories but **gallery** seems to be more interesting to me.

navigating to gallery dir, I found many files and directories,

![image](https://user-images.githubusercontent.com/67465230/156874071-6889b181-87f9-4432-8bf6-b8e9f0836192.png)

seems like **gconfig.php** stands high above from others :). Let's view the content of this file,

```bash
cat gconfig.php
```

![image](https://user-images.githubusercontent.com/67465230/156874075-e931d096-55e7-42f7-b651-055822b68bf2.png)

Okay, we got the hardcoded credentials and I believe these credentials are for **phpMyAdmin** (which we found on nikto scan). Let's visit http://10.0.2.68/phpmyadmin,

we got this login page and we'll enter the creds we found in pconfig.php file,

![image](https://user-images.githubusercontent.com/67465230/156874082-0fa4209f-7dc0-4263-a7e1-e0c246289d4b.png)

BOOM!! we're inside the phpmyadmin login page,

![image](https://user-images.githubusercontent.com/67465230/156874097-37ee3eb1-d7c6-41fc-bed6-a33d8dbedb13.png)

after enumerating website a little bit, **gallery** tab seems interesting and after navigating to it, there seems **dev_accounts** which is our target,

![image](https://user-images.githubusercontent.com/67465230/156874106-fc191dea-14e4-4478-97d7-8d52c9b84ea9.png)

there are the username and passwords and we can see them by navigating them by switching to **browse** tab,

![image](https://user-images.githubusercontent.com/67465230/156874112-9c0d6530-3d31-4571-8edf-d27a7a1f58e9.png)

we got username and password's hashes of **loneferret** and **dreg** user.

Let's now take time to crack the passwords. We have 2 methods to crack the passwords, i.e. **Hydra** and **Hashcat** but I'll use only hydra tool here.

Let's see how we can crack the password by brute forcing the password using Hydra tool,

```bash
hydra -l loneferret -P /usr/share/wordlists/fasttrack.txt 10.0.2.68 ssh -e nsr -f -t 4

the parameters here used are: 
-l (login name), -P (Password list), IP (machine IP), ssh (service), -e (nsr = n: null password, s: login and pass, and r: reversed login), -f (exit after login/password found), -t 4 (parallel number of connected tasks)
```

![image](https://user-images.githubusercontent.com/67465230/156874219-2d46ac95-5f3e-4c78-b3ff-bacaa3bd73c5.png)

our password for user is **loneferret**:**starwars**. 

Let's get into system via SSH as loneferret user,

```bash
ssh loneferret@10.0.2.68
```

![image](https://user-images.githubusercontent.com/67465230/156874223-f7b159ec-aa7a-4a6b-9d7e-c1fa3ad843ec.png)

we got access to loneferret user.

Let's now check if we can run any binary as sudo to elevate our privileges,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/156874225-deaeaeb1-8e6c-4c56-b6df-2ec50419bee3.png)

one 2nd line, we can ht binary as sudo without providing any password. 

we when binary,

```bash
sudo /usr/local/bin/ht
```

![image](https://user-images.githubusercontent.com/67465230/156874228-5b0ccd3f-398b-4539-87e0-f096f5786387.png)

we got this term-256color error. Let's first issue a command to export this xterm,

```bash
export TERM=xterm
```

![image](https://user-images.githubusercontent.com/67465230/156874230-fa299f16-7081-4306-b888-21b869bd32e8.png)

now when we run the command `sudo /usr/local/bin/ht` again,

![image](https://user-images.githubusercontent.com/67465230/156874239-0e0810b0-c4e4-40e3-8770-f20cc60b4802.png)

The **ht** editor opens within the terminal window. At this point, I should be able to open any file with root permissions.

Using **Alt+F**, we'll open the file menu,

![image](https://user-images.githubusercontent.com/67465230/156874242-fde5d136-d4d2-40d2-982f-3305c6296d3e.png)

then press enter on Open,

type **/etc/sudoers**,

![image](https://user-images.githubusercontent.com/67465230/156874245-10ca6f6e-176f-49bc-8c6e-28084bd7dfec.png)

and press enter.

We'll get default sudoers file,

![image](https://user-images.githubusercontent.com/67465230/156874247-244b1659-71b8-45a7-af30-e14d1812e48a.png)

we can see that user loneferret already had a line item with root privileges to two commands that doesn't require password entry.

In order to get system access, let's add **/bin/bash** directly after **/usr/local/bin/ht**,

![image](https://user-images.githubusercontent.com/67465230/156874250-d7e4716d-da3d-4cbb-9602-a651e48a7789.png)

this will provide root access to **bash** shell. Close this file after saving it. Let's run the command,

```bash
sudo /bin/bash
```

![image](https://user-images.githubusercontent.com/67465230/156874252-40ee2844-36b0-4955-b650-d7f084d65c87.png)

and we got ROOT!!

Let's navigate to the root directory and get the flag,

![image](https://user-images.githubusercontent.com/67465230/156874265-faa7d990-85af-43eb-a070-9eda2d8211f6.png)
