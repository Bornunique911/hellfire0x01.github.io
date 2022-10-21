---
layout: post
title: "Vulnversity"
date: 2022-10-21
categories: [Tryhackme, Easy-THM]
tags: [dirsearch, file-upload, burpsuite, reverse-shell, SUID]
image: ../../assets/img/posts/vulnversity.png 

---

## Description

Learn about active recon, web app attacks and privilege escalation.

|**Room**|[Vulnversity](https://tryhackme.com/room/vulnversity)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[tryhackme](https://tryhackme.com/p/tryhackme)|

---

Let's start with port scanning which can be done with nmap,

```bash
nmap -sV -sC 10.10.59.93 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/183449631-e9b8bd4a-d9f6-4ec1-84e8-306c77424d98.png)

We got several ports open but port 22 depicts that we can ssh into machine and port 3333 will take us to website. Let's enumerate port 3333.

Visiting http://10.10.59.93:3333,

![image](https://user-images.githubusercontent.com/67465230/183449708-f201459c-2066-4f44-944a-7b58a56359c0.png)

we can see the home page of the website. Let's do a bit enumeration but after sometime, I found nothing so I decided to brute force hidden directories using dirsearch,

```bash
python3 /home/kali/tools/dirsearch/dirsearch.py -u http://10.10.59.93:3333 -e txt,php -i 200,301 -o dirsearch.py
```

![image](https://user-images.githubusercontent.com/67465230/183449749-43cfeaf9-4828-4afa-ac49-a850ddacd631.png)

we got some directories and enumerating each one of them doesn't give us the fruit we want except **/internal** directory, which seems the fruit we want. Let's visit,

![image](https://user-images.githubusercontent.com/67465230/183449964-68620756-d9da-4d24-b039-8047715657c7.png)

Seems like this is a page where we can upload our files.

Let's start the burpsuite and enable the intercept mode so that when we submit the request, it intercepts the request and we can inspect it.

Let's browse the file we want to upload,

![image](https://user-images.githubusercontent.com/67465230/183450009-dd16b39f-72fe-4452-93b5-9bf74f3c8f1e.png)

and hitting Submit button will straightaway send the request to burp,

![image](https://user-images.githubusercontent.com/67465230/183450058-08da0c0d-525b-4b93-a28d-a387c2102ac9.png)

We can see that we've uploaded file.php file. Let's send this request to repeater to inspect it.

When we send this request with repeater,

![image](https://user-images.githubusercontent.com/67465230/183450127-0ffeaaef-a888-49d7-b59a-8c9d111c2af3.png)

we got response that "Extension not allowed", means that the file we've uploaded with .php extension is not allowed as webserver is filtering of blacklisting .php extension.

Now, that we know this website is filtering .php extension, that means we manually have to try all extensions one by one to check which extension is allowed. Hopefully, burp intruder can do this job (Automation!!).

Sending this request to intruder,

![image](https://user-images.githubusercontent.com/67465230/183450179-f3c98527-5666-4de7-b639-e89667db38b9.png)

We'll clear all mark and then select specific part which we have to fuzz in order to find correct extension and then add mark.

Let's create a payload list and uncheck the URL encode box so that when payloads are thrown at intruder's mark area, they won't get encoded,

![image](https://user-images.githubusercontent.com/67465230/183450237-2fe45cfd-2969-449a-b9c7-668bb6deb8ee.png)

Finally, let's start the attack,

![image](https://user-images.githubusercontent.com/67465230/183450293-f4a141c5-71d3-4e3a-a9d6-44ccd6188034.png)

and there we found a .phtml extension which has different length. Let's check it's response,

When sending this .phtml extension request,

![image](https://user-images.githubusercontent.com/67465230/183450343-a7f3d828-943d-4a69-a623-3ac68eaa4a7d.png)

we got render response which says "Success" means we can upload only .phtml extension files.

We'll download a php reverse shell from https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php and then rename it with .phtml extension.

Now, let's edit our php reverse shell and replace IP with our tun0 IP and port to desired one,

![image](https://user-images.githubusercontent.com/67465230/183450395-d90366f5-63b7-4cfc-a62c-519a765e77c8.png)

save this and exit.

Now, Let's start the netcat listener using `nc -nvlp 4444` and now we'll upload our shell on website, when executed, netcat listening on port 4444 will catch connection and we'll be presented with shell.

Browse php shell and click on Submit button,

![image](https://user-images.githubusercontent.com/67465230/183450454-ccfe12b2-fd52-4793-9abb-c0e759b75765.png)

our shell is successfully uploaded but we didn't get a shell. Why? We don't get our shell because we haven't executed the uploaded shell from website. We have just uploaded it.

So, let's visit http://10.10.59.93:3333/internal/uploads

![image](https://user-images.githubusercontent.com/67465230/183450517-18364842-9888-4619-a81d-46b25382cb6c.png)

and clicking on the file we uploaded,

We'll get a shell.

![image](https://user-images.githubusercontent.com/67465230/183450572-6652fe50-ebc2-455c-af7b-f355366c5787.png)

But this shell is underprivilege, let's make it for friendly,

```bash
/bin/bash -i
```

![image](https://user-images.githubusercontent.com/67465230/183450626-6f47a998-3e31-4775-932f-d3afdedf25fd.png)

now this shell looks cool.

Let's navigate to /home directory for user flag,

![image](https://user-images.githubusercontent.com/67465230/183450664-bbbac3b2-6cb5-4e97-88d0-1d70f5386674.png)

there we found a user.txt file. 

Let's come the privilege escalation part. Now we need to look for the SUID Binaries.

SUID is a file permission which is added to/given to few of the binaries which are allowed to be run by the user, but they run under the name of their owner i.e. test.bin when having SUID permissions set on root when ran on under the "billy" account will be run under root.

```bash
find / -perm -04000 -type f -ls 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/183450722-1524249b-9699-4b5c-bac5-7a8b77a5822e.png)

we got a binary which has SUID bit set on it named as **/bin/systemctl**.

let's check the permissions of /bin/systemctl file,

```bash
ls -la /bin/systemctl
```

![image](https://user-images.githubusercontent.com/67465230/183450787-923fcbd9-4276-47f8-b953-630ef37bfeba.png)

**systemctl** is a binary that controls interfaces for init systems and service managers. Remember making your services run using the systemctl command during the boot time. All those tasks are handled as units and are defined in unit folders. By default systemctl will search these files in **/etc/system/systemd**.

For this machine we do not have access to the paths owned by root and by so we can't made the unit file. Although we can set **environment variables**. So let's do the PrivEsc.

```
Reference: https://gtfobins.github.io/gtfobins/systemctl/#suid
```

The first thing we need to is create an environment variable! 

```bash
TF=$(mktemp).service
```

![image](https://user-images.githubusercontent.com/67465230/183450878-117d6d0a-713c-4374-8592-d8d3d7b05fe7.png)

Now we need to create a unit file and assign this to the environment variable.

```bash
echo '[Service]
> ExecStart=/bin/bash -c "cat /root/root.txt > /tmp/flag"
> [Install]
> WantedBy=multi-user.target' > $TF
```

![image](https://user-images.githubusercontent.com/67465230/183450943-702704cc-2f7e-4916-bcd8-b0aade1d0f70.png)

What we have done here is to simply create a service which will be executing "BASH", then reading the flag from the root directory and then writing it in the flag (file) in /opt directory.

Now we need to run this unit file using systemctl.

```bash
/bin/systemctl link $TF
```

![image](https://user-images.githubusercontent.com/67465230/183451005-99b94e99-f8d9-435d-8afe-37ce2146f92e.png)

```bash
/bin/systemctl enable --now $TF
```

![image](https://user-images.githubusercontent.com/67465230/183451084-3bf68c94-5d14-44de-9fea-476f9aa2d5e2.png)

Now we can find the "flag" file in the / directory containing the flag! 

![image](https://user-images.githubusercontent.com/67465230/183451178-f3cd69d8-830c-4527-a2bb-65361cbffe16.png)

there we have our flag. 