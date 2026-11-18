# Payday (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal that SSH is running on port 22, an Apache web server is running on port 80, SMB is running on port 445, ports 110 and 995 are used for POP3, and ports 143 and 993 are used for IMAP.
```
┌──(root㉿shushan)-[/home/shushan/OffSec/Labs/Payday]
└─# nmap -p- -sV -T4 192.168.226.39 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-25 21:12 +0300
Nmap scan report for 192.168.226.39
Host is up (0.21s latency).
Not shown: 65527 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.6p1 Debian 5build1 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.4 ((Ubuntu) PHP/5.2.3-1ubuntu6)
110/tcp open  pop3        Dovecot pop3d
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: MSHOME)
143/tcp open  imap        Dovecot imapd
445/tcp open  netbios-ssn Samba smbd 3.0.26a (workgroup: MSHOME)
993/tcp open  ssl/imap    Dovecot imapd
995/tcp open  ssl/pop3    Dovecot pop3d
<SNIP>
```

The web application on port 80 is a CS-Cart international shop.
![](assets/payday/Pasted%20image%2020261111175258.png)

Searchsploit displays a local file inclusion vulnerability affecting CS-Cart 1.3.3 (CVE-2006-2863).
![](assets/payday/Pasted%20image%2020261111175349.png)

The proof of concept provides the affected URL paths to exploit this vulnerability.
![](assets/payday/Pasted%20image%2020261111175800.png)
## Initial Access
While the exact version of the CS-Cart installation was not identified through the web UI, the target is found to be vulnerable. The local file inclusion vulnerability is then exploited to read `/etc/passwd`, revealing a local `patrick` user.
![](assets/payday/Pasted%20image%2020261111175648.png)

An SSH login attempt using `patrick` as both the username and password is successful, granting access to the `payday` host.
![](assets/payday/Pasted%20image%2020261111175903.png)

The local flag is found in `patrick`'s home directory.
![](assets/payday/Pasted%20image%2020261111180050.png)
## Privilege Escalation
The `sudo -l` command reveals that `patrick` can execute any command using `sudo`.
![](assets/payday/Pasted%20image%2020261111180151.png)

The `sudo su` command is used to switch to the `root` account and retrieve `/root/proof.txt`.
![](assets/payday/Pasted%20image%2020261111180227.png)

## Finding & Remediation
### CS-Cart File Inclusion (CVE-2006-2863)
#### Description
The target runs a vulnerable CS-Cart installation affected by CVE-2006-2863. The vulnerability allows unauthenticated attackers to manipulate the `classes_dir` parameter in `class.cs_phpmailer.php` to include arbitrary files.
#### Impact
An unauthenticated attacker can exploit this vulnerability to read sensitive files from the target system. In this case, the vulnerability was used to retrieve `/etc/passwd` and identify the local `patrick` user.
#### Remediation
Replace the outdated CS-Cart installation with a currently supported version, such as CS-Cart 4.21.2 or later. Ensure that user-controlled parameters cannot be used to include arbitrary files.
### Use of Weak Credentials
#### Description
The local `patrick` account uses a weak password identical to its username (`patrick`), allowing the credentials to be guessed easily.
#### Impact
An unauthenticated attacker can guess the weak password and gain SSH access to the target system as the `patrick` user.
#### Remediation
Enforce strong, unique passwords for system accounts and prohibit passwords identical to usernames. Implement SSH authentication protections such as rate limiting and account lockout policies, and consider using key-based authentication.
### Sudo Misconfiguration
#### Description
The `patrick` user is granted unrestricted `sudo` privileges through the `(ALL) ALL` configuration, allowing the execution of any command as `root` after providing the user's password.
#### Impact
An attacker who compromises the `patrick` account and obtains its password can execute arbitrary commands with `root` privileges, resulting in complete system compromise.
#### Remediation
Remove unrestricted `sudo` privileges from the `patrick` account. Grant only the specific administrative commands required for legitimate tasks and follow the principle of least privilege when configuring sudoers rules.
