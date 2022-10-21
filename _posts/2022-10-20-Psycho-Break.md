---
layout: post
title: "Psycho Break"
date: 2022-10-20
categories: [Tryhackme, Easy-THM]
tags: [rustscan, atbash-cipher, ffuf, reverse-image-search, Command-Injection, hashcat, morse-audio, steghide, ftp, multi-tap-cipher, /etc/crontab, /etc/passwd]
image: ../../assets/img/posts/psychobreak.png 

---

## Description

Help Sebastian and his team of investigators to withstand the dangers that come ahead.

|**Room**|[Psycho Break](https://tryhackme.com/room/psychobreak)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Easy|
|**Creator**|[shafdo](https://tryhackme.com/p/shafdo)|

---

Starting off with deploying the machine and quickly scanning the open ports with rustscan,

```bash
rustscan -a 10.10.146.46 --ulimit 5000
```

![image](https://user-images.githubusercontent.com/67465230/186803679-7f2fe2ee-bbdf-4ded-aa3e-21653f5ecc4c.png)

we got some ports open. Let's scan them in detail using nmap,

```bash
nmap -sC -sV -p21,22,80 10.10.146.46 -oN nmap.txt
```

![image](https://user-images.githubusercontent.com/67465230/186803715-3bca9b6f-199a-49da-8338-3a068792cca4.png)

Result scan shows that port 21 is running ftp service, port 22 is running ssh service, port 80 is running apache webserver. Let's start enumerating port 80.

Visiting http://10.10.146.46,

![image](https://user-images.githubusercontent.com/67465230/186803756-31b32ca7-6375-45af-b4ba-c97a8ed12a39.png)

we got a webpage showing a message that "All begins From Here", so this means that it will be a long journey from here. _Let's start!_

Looking at the source code of the webpage,

![image](https://user-images.githubusercontent.com/67465230/186805730-dde7985f-5634-4a45-9f67-b3828d581628.png)

there is a comment mentioning a name of a person (username enumeration?) and a hidden path name **/sadistRoom**.

Following this path http://10.10.146.46/sadistRoom,

![image](https://user-images.githubusercontent.com/67465230/186805792-62c33a31-ba59-45df-95d7-8290c19ce72a.png)

we got a disturbing picture.

Let's look at the source code,

![image](https://user-images.githubusercontent.com/67465230/186803798-6246dd0b-a6bb-47a1-b8db-06983212e079.png)

there is a script.js file and I decided to inspect it,

![image](https://user-images.githubusercontent.com/67465230/186803885-d90bb8d2-ec5a-459a-998f-f0f5fdc88145.png)

we got the locker room key.

Now, we can access locker room,

![image](https://user-images.githubusercontent.com/67465230/186803937-f725a776-3fd0-4d52-82ba-346703518705.png)

there is a key which is encoded.

Let's try to decode this key using [atbash tool](http://rumkin.com/tools/cipher/atbash.php)

![image](https://user-images.githubusercontent.com/67465230/197121620-adb69858-4684-469b-9410-4fb578009416.png)

our key is cracked.

Now, let's visit the http://10.10.146.46/map and enter the decoded key, 

![image](https://user-images.githubusercontent.com/67465230/186803993-45256e1e-8c14-4d7c-90b5-77371268a8ac.png)

We got in! We can now see the new paths which we have to cover yet,

![image](https://user-images.githubusercontent.com/67465230/186804022-03c6d2a8-797b-446c-b063-bed51affdaea.png)

Navigate to Safe Heaven, http://10.10.146.46/SafeHeaven,

![image](https://user-images.githubusercontent.com/67465230/186804044-6378528f-5ee5-4798-a3f4-8a8915bed19a.png)

We can see many pictures here but not useful. 

Let's take a look at source code,

![image](https://user-images.githubusercontent.com/67465230/186804082-19eb7bff-217c-4b53-ad79-98653ffb8ca2.png)

In comment, there is a hint for use to find something (_which we can't see on surface.. might be directory busting?_)

Using ffuf tool, we can find hidden directory,

```bash
ffuf -u http://10.10.146.46/SafeHeaven/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -recursion 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/186805954-39842baf-33df-48b0-a01f-930b87feb0db.png)

we got a path /keeper. Let's enumerate it.

Visit http://10.10.146.46/SafeHeaven/keeper,

![image](https://user-images.githubusercontent.com/67465230/186804125-f956d9e7-b012-41bc-a732-80afc9afed41.png)

And we meet the keeper. There is button at the bottom of the picture.

Clicking on Escape Keeper button, we are presented with a webpage with time ticking off and there is a picture of staircase,

![image](https://user-images.githubusercontent.com/67465230/186804149-2762b0a4-4974-48e9-8629-9f408fc7a21d.png)

The message is saying that we need to enter the real location in image before time runs out.

Taking a look at source code and we get to know that we need to use Google Reverse Image on this picture,

![image](https://user-images.githubusercontent.com/67465230/186804177-ad2f7e59-0460-4744-b48f-e781dc0c4a05.png)

After using Google Reverse Image on this picture,

![image](https://user-images.githubusercontent.com/67465230/186804214-0afb5a62-190b-409e-bc73-7fd58d95e27d.png)

the name of the place is St. Augustine Lighthouse.

After pasting the name of the place,

![image](https://user-images.githubusercontent.com/67465230/186804285-01df31db-8012-45a9-9e0c-af0a2530dede.png)

we get the key.

Now, navigating to http://10.10.146.46/abandonedRoom and pasting the key,

![image](https://user-images.githubusercontent.com/67465230/186804319-15da7a58-84ca-40c1-b348-52665fdba2a0.png)

we got in a room which has some disturbing picture,

![image](https://user-images.githubusercontent.com/67465230/186804436-ba7f9370-d579-4aa2-aa3c-26cb0a71a2a4.png)

it has button of "Go Further". 

Let's _Go Further_,

![image](https://user-images.githubusercontent.com/67465230/186804483-8092b124-be7e-41f5-a7ef-61d0b13d5dd2.png)

and we will be presented with a spider attacking us _gif_.

Let's take a look at source code,

![image](https://user-images.githubusercontent.com/67465230/186804515-460fc59e-fed3-430b-b664-5498ef13ca75.png)

A comment says there is a _shell_ on current page (_command injection?_)

Trying command injection on this page,

```bash
?shell=whoami
```

![image](https://user-images.githubusercontent.com/67465230/186804565-338b2222-2b0d-4202-bde2-67f9d3512f4f.png)

Message shows that the command `whoami` is not permitted.

Let's use another command to list directory content,

```bash
?shell=ls
```

![image](https://user-images.githubusercontent.com/67465230/186804598-1611cc91-f4e8-40ab-95f8-67ac498c2e4c.png)

there are some files listed out. After taking a look at them, there is nothing much about them.

Getting one directory back,

```bash
?shell=ls ..
```

![image](https://user-images.githubusercontent.com/67465230/186804628-dc1fc267-cffb-489e-a5a9-18c9875164e4.png)

we got the hashes and 1 index file.

Let's use hashcat tool to crack these hashes,

```bash
hashcat.exe -m 0 crack.txt rockyou.txt -O
```

![image](https://user-images.githubusercontent.com/67465230/186804648-fe8cee80-88aa-44ce-8393-fa590ea4d1b3.png)

Our hashes has been cracked and we got our password.

Credentials - `laura:********`

One of these hashes has directory feature enabled, so visiting it,

![image](https://user-images.githubusercontent.com/67465230/186804678-2e50b18b-9839-4bd5-a537-f52927c95193.png)

we got 2 files in there. Let's download them on our system.

Unzipping the zip file,

```bash
unzip helpme.zip
```

![image](https://user-images.githubusercontent.com/67465230/186804720-b2803657-7cfd-4d86-94d4-2bf113773b60.png)

it extracted 2 files out from here.

Reading the text file,

![image](https://user-images.githubusercontent.com/67465230/186804755-f3be3857-e5bd-4329-87ee-280fce196ea2.png)

We got the message from joseph (username enumeration!) that we need to save him from Ruvik.

Now taking a look what jpg file type is,

```bash
file Table.jpg
```

![image](https://user-images.githubusercontent.com/67465230/186804787-43d5462a-124f-467d-ab0f-fc1c36014355.png)

it is a zip file. 

So, rename file with zip extension and then extract it,

```bash
mv Table.jpg table.zip
unzip table.zip
```

![image](https://user-images.githubusercontent.com/67465230/186804825-e8d21d3c-f870-4841-be0b-5b9a6b32d8e9.png)

After listening to audio we got, a "beep" sounds can be heard from which I can't comprehend anything. 

So I decided to use [Morse audio decoder](https://morsecode.world/international/decoder/audio-decoder-adaptive.html),

![image](https://user-images.githubusercontent.com/67465230/186804977-6de3eb76-8830-4e86-9cd4-33449036cc6e.png)

we got our key!

Now, extracting hidden data from image using steghide,

```bash
steghide extract -sf Joseph_Oda.jpg
```

![image](https://user-images.githubusercontent.com/67465230/186805019-855ec007-0403-4a4b-a20b-984bf43bb553.png)

we got a thankyou.txt file.

![image](https://user-images.githubusercontent.com/67465230/186805045-4290903b-b05d-477d-a7ef-1f6177d1bf3d.png)

we got another message from joseph which details our FTP credentials.

Now, logging into FTP service using the credentials we get,

```bash
ftp 10.10.146.46
```

![image](https://user-images.githubusercontent.com/67465230/186805067-8434a7c9-1a91-4783-8448-dcd87eba4aa8.png)

Now, enumerating directory content,

![image](https://user-images.githubusercontent.com/67465230/186805112-a7b648e5-e60b-4435-9118-66aa2436f99a.png)

we got 2 files program and random.dic.

Let's switch to binary mode to transfer these 2 files on our system,

```bash
binary
get randon.dic
get program
```

Looking at random.dic file content, I got that file contents which can be used for bruteforcing.

I tried running this program but it didn't worked so I give this program executable permission and then ran this program and it worked,

![image](https://user-images.githubusercontent.com/67465230/186805172-8d82b280-a238-442f-8c87-0ebca7b8adda.png)

```bash
chmod +x program
./program
```

![image](https://user-images.githubusercontent.com/67465230/186805215-6fa57c7b-16c4-4e5f-8e6f-ecc3af283dbc.png)

So, after running this program, I get the idea that I need to insert one value from random.dic file in order to break this program and get the correct information out this program.

So, I found a little python script to automate this cracking process,

```python
import os
import subprocess
import sys

f = open("random.dic", "r")

keys = f.readlines()

for key in keys:
        key = str(key.replace("\n", ""))
        print (key)
        subprocess.run(["./program", key])
```

After writing this script, we can run this with python3,

```bash
python3 script
```

![image](https://user-images.githubusercontent.com/67465230/186805253-bfda2390-6970-4e00-8a43-ad065ed497a1.png)

after sometime, we will see that it is cracked. But again, it is all the numbers we can see, nothing else.

Let's try to crack this code using [Multi-Tap Cipher](https://www.dcode.fr/multitap-abc-cipher),

![image](https://user-images.githubusercontent.com/67465230/186805278-4c08cd44-55d1-4305-9787-6ef8354a82fe.png)

we got our plain text password.

So, trying to login via ssh using the credentials I found,

```bash
ssh kidman@10.10.149.118
```

![image](https://user-images.githubusercontent.com/67465230/186805319-d4bc1b7d-fa5b-4330-b7da-dc91a164ace4.png)

and I got in.

Enumerating directory and we got our user flag,

![image](https://user-images.githubusercontent.com/67465230/186805364-353ebbf7-e384-4048-b9c4-242f4d0e625b.png)

and, there is another file named .the_eye.txt and .readThis.txt file.

Reading .the_eye.txt file,

![image](https://user-images.githubusercontent.com/67465230/186805380-c51b09e1-008e-4ecb-bbcd-2653243b838c.png)

It says someone is watching us.

And, Reading .readThis.txt file,

![image](https://user-images.githubusercontent.com/67465230/186805421-332f14fe-10be-4484-a7bc-a6799e6aeb87.png)

we got chunk of data which is encoded in ROT47.

So we can use [CyberChef](https://gchq.github.io/CyberChef/) to decode this data,

![image](https://user-images.githubusercontent.com/67465230/186805450-137e5567-6208-44a0-acbd-29cf632b3ebf.png)

We need to search for the specified string and after searching through it, I get the path of it.

Let's check in `/etc/crontab` if this file runs periodically,

```bash
cat /etc/crontab
```

![image](https://user-images.githubusercontent.com/67465230/186805489-36813888-fabf-40ad-93b2-9263346b86df.png)

Yes, it does run every 2 minutes as root.

Reading the content of file,

![image](https://user-images.githubusercontent.com/67465230/186805509-c3671063-ad24-4661-a840-11d7a2883317.png)

this is a script which does nothing specifically.

Let's modify this script to actually copy the content of root.txt file into .the_eye.txt file,

```bash
cat > /var/.the_eye_of_ruvik.py
#!/usr/bin/python3

import subprocess
import random

stuff = ["I am watching you.","No one can hide from me.","Ruvik ...","No one shall hide from me","No one can escape from me"]
sentence = "".join(random.sample(stuff,1))
subprocess.call("cat /root/root.txt > /home/kidman/.the_eye.txt",shell=True)
```

after waiting for 2 minutes, the content of root.txt file (root flag) get's copied in .the_eye.txt file.

Now, we can look at /etc/passwd file,

![image](https://user-images.githubusercontent.com/67465230/186805547-e2be7693-d914-4db2-a922-85398772dabb.png)

we can see ruvik user. We can delete this user (_optional_).

Modifying the script a little bit,

```bash
cat > /var/.the_eye_of_ruvik.py 
#!/usr/bin/python3 

import subprocess 
import random 

stuff = ["I am watching you.","No one can hide from me.","Ruvik ...","No one shall hide from me","No one can escape from me"] 
sentence = "".join(random.sample(stuff,1)) 
subprocess.call("userdel ruvik",shell=True)
```

![image](https://user-images.githubusercontent.com/67465230/186805583-bbc9fffe-511e-4aab-ade2-dd46b0877cf6.png)

after 2 minutes, user ruvik gets deleted from the system.