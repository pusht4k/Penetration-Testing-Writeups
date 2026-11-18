# Seppuku (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan finds eight open ports on the target, with FTP running on port 21, SSH on port 22, an Nginx web server on port 80, Samba SMB on ports 139 and 445, LiteSpeed on ports 7080 and 8088, and an Apache web server running on port 7601.
```
┌──(root㉿shushan)-[/home/shushan/OffSec/Labs]
└─# nmap -p- -sV -T4 192.168.135.90
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-26 20:02 +0300
Nmap scan report for 192.168.135.90
Host is up (0.090s latency).
Not shown: 65527 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           vsftpd 3.0.3
22/tcp   open  ssh           OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http          nginx 1.14.2
139/tcp  open  netbios-ssn   Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn   Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
7080/tcp open  ssl/empowerid LiteSpeed
7601/tcp open  http          Apache httpd 2.4.38 ((Debian))
8088/tcp open  http          LiteSpeed httpd
<SNIP>
```

A `gobuster` directory brute-force scan of the web application running on port 7601 finds a large number of accessible directories, the most notable ones being `keys` and `secret`.
![](assets/seppuku/Pasted%20image%2020261115191242.png)

The `keys` directory has directory listing enabled and contains two files, `private` and `private.bak`.
![](assets/seppuku/Pasted%20image%2020261115191319.png)

Both files contain private SSH RSA keys.
![](assets/seppuku/Pasted%20image%2020261115191358.png)

The `secret` directory also has directory listing. The directory contains the `passwd.bak` and `shadow.bak` files that are seemingly backups of the `/etc/passwd` and `/etc/shadow` files on Linux.
![](assets/seppuku/Pasted%20image%2020261115191451.png)

Both files contain the `rabbit-hole` user, indicating that this is a rabbit hole.
![](assets/seppuku/Pasted%20image%2020261115191601.png)
## Initial Access
Null SMB authentication with Enum4Linux is used to enumerate local users on the target, revealing the `seppuku`, `samurai` and `tanto` users.
![](assets/seppuku/Pasted%20image%2020261115191642.png)

SSH authentication using the previously obtained RSA key is attempted for all three users. The key is accepted for authentication as `tanto`.
![](assets/seppuku/Pasted%20image%2020261115191735.png)
## Privilege Escalation
The home directory of the `seppuku` user contains a `.passwd` file that is readable by all users on the `seppuku` host.
![](assets/seppuku/Pasted%20image%2020261115191847.png)

The file seemingly contains a password. However, it does not contain a username.
![](assets/seppuku/Pasted%20image%2020261115191925.png)

NetExec is used to spray the password for both the `seppuku` and `samurai` users, revealing that it is used by the latter.
![](assets/seppuku/Pasted%20image%2020261115192007.png)

The `samurai` user is permitted to execute the following sudo command without providing a password.
```
/../../../../../../home/tanto/.cgi_bin/bin /tmp/*
```
![](assets/seppuku/Pasted%20image%2020261115192106.png)

To abuse this misconfiguration, the initial shell as `tanto` is used to create a `rev.sh` file in the `/tmp` directory containing a Bash reverse shell one-liner. The `bin` file in the specified path shown in the `sudo` command is replaced with a copy of `bash`. 
This will effectively execute the equivalent to `sudo /bin/bash /tmp/rev.sh` when the command is used.
![](assets/seppuku/Pasted%20image%2020261115192322.png)

The `sudo` command is then used to execute `/tmp.rev.sh`.
![](assets/seppuku/Pasted%20image%2020261115192542.png)

A reverse shell running as `root` is established right after.
![](assets/seppuku/Pasted%20image%2020261115192603.png)
## Findings & Remediation

### Vulnerability 1
#### Explanation
#### Impact
#### Remediation
