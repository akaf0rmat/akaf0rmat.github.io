---
layout: post
title: HTB - Starting Point - Fawn
author: _f0rmat
date: 2025-08-14
tags: StartingPoint HackTheBox Tier0 FTP 
category: CTF
---

```
Difficulty: Very Easy
Status: Complete
IP: 10.129.61.74
Note: As these machines are Task Based, the formatting of the blog post has been changed, each of these starting point machines on their various tiers are quite similar and designed to give players and learners a solid foundation to work up from before taking down their initial CTF boxes.
```

## Improved skills
- nmap
- enumeration
- FTP

## Used tools
- nmap
- google

---

# Task 1

What does the 3-letter acronym FTP stand for? 

```bash
File Transfer Protocol
```

# Task 2

Which port does the FTP service listen on usually? 

```bash
21
```

# Task 3

FTP sends data in the clear, without any encryption. What acronym is used for a later protocol designed to provide similar functionality to FTP but securely, as an extension of the SSH protocol? 

```bash
SFTP
```

# Task 4

What is the command we can use to send an ICMP echo request to test our connection to the target? 

```bash
ping
```

# Task 5

From your scans, what version is FTP running on the target? 

```bash
vsFTPd 3.0.3
```

# Task 6

From your scans, what OS type is running on the target? 

```bash
unix
```

# Task 7

What is the command we need to run in order to display the 'ftp' client help menu? 

```bash
ftp -?
```

# Task 8

What is username that is used over FTP when you want to log in without having an account? 

```bash
anonymous
```

# Task 9

What is the response code we get for the FTP message 'Login successful'? 

```bash
230
```

# Task 10

There are a couple of commands we can use to list the files and directories available on the FTP server. One is dir. What is the other that is a common way to list files on a Linux system. 

```bash
ls
```

# Task 11

What is the command used to download the file we found on the FTP server?  

```bash
get
```

# Submit Flag

Submit root flag 

```bash
035db21c881520061c53e0536e44f815
```