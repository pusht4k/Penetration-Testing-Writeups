# Stapler (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show FTP running on port 21 with anonymous login enabled, SSH running on port 22, a `tcpwrapped` service on port 53, a PHP web server running on port 80, Samba running on port 139, MySQL running on port 3306 and an Apache web server running on port 12380. Nmap identifies the service on port 666 as `pkzip-file`, with the fingerprint output containing the filename `message2.jpg` alongside seemingly unreadable strings, suggesting that the service returns a ZIP archive containing an image.
```
┌──(root㉿shushan)-[/home/shushan/OffSec/Labs/Stapler]
└─# nmap -p- -A -T4 192.168.233.148
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-27 22:23 +0300
Nmap scan report for 192.168.233.148
Host is up (0.083s latency).
Not shown: 65523 filtered tcp ports (no-response)
PORT      STATE  SERVICE     VERSION
21/tcp    open   ftp         vsftpd 2.0.8 or later
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.45.207
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
22/tcp    open   ssh         OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 81:21:ce:a1:1a:05:b1:69:4f:4d:ed:80:28:e8:99:05 (RSA)
|   256 5b:a5:bb:67:91:1a:51:c2:d3:21:da:c0:ca:f0:db:9e (ECDSA)
|_  256 6d:01:b7:73:ac:b0:93:6f:fa:b9:89:e6:ae:3c:ab:d3 (ED25519)
53/tcp    open   tcpwrapped
80/tcp    open   http        PHP cli server 5.5 or later
|_http-title: 404 Not Found
139/tcp   open   netbios-ssn Samba smbd 4.3.9-Ubuntu (workgroup: WORKGROUP)
666/tcp   open   pkzip-file  .ZIP file
| fingerprint-strings: 
|   NULL: 
|     message2.jpgUT 
|     QWux
|     "DL[E
|     #;3[
|     \xf6
|     u([r
|     qYQq
|     Y_?n2
|     3&M~{
|     9-a)T
|     L}AJ
|_    .npy.9
3306/tcp  open   mysql       MySQL 5.7.12-0ubuntu1
| mysql-info: 
|   Protocol: 10
|   Version: 5.7.12-0ubuntu1
|   Thread ID: 7
|   Capabilities flags: 63487
|   Some Capabilities: Speaks41ProtocolOld, ODBCClient, DontAllowDatabaseTableColumn, Support41Auth, SupportsTransactions, SupportsCompression, IgnoreSpaceBeforeParenthesis, LongPassword, IgnoreSigpipes, SupportsLoadDataLocal, InteractiveClient, Speaks41ProtocolNew, ConnectWithDatabase, LongColumnFlag, FoundRows, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: KJ\x07CB#D;tw{|*~/v{}"\x17
|_  Auth Plugin Name: mysql_native_password
12380/tcp open   http        Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Tim, we need to-do better next year for Initech
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

The FTP banner reveals a potential user, `Harry`. In addition a `note` file is found in the root FTP directory.
![](assets/stapler/Pasted%20image%2020261115165958.png)

The file contains two additional names, Elly and John, which could also be used as potential usernames later.
![](assets/stapler/Pasted%20image%2020261115170027.png)

Port 666 is accessed through Netcat. The output is redirected to a file.
![](assets/stapler/Pasted%20image%2020261115170127.png)

The archive contains an image mentioning Scott.
![](assets/stapler/Pasted%20image%2020261115170159.png)

The web application's page source on port 12380 reveals another potential user - `Tim`.
![](assets/stapler/Pasted%20image%2020261115170254.png)

Enum4Linux finds a `kathy` share that is accessible through null authentication.
![](assets/stapler/Pasted%20image%2020261115170401.png)

In addition, Enum4Linux displays all system users.
![](assets/stapler/Pasted%20image%2020261115170443.png)
## Initial Access
All usernames are saved to a file. The `hydra` tool is then used to perform a brute force attack, revealing that the user `elly` uses her username backwards as her password (`ylle`) in FTP.
![](assets/stapler/Pasted%20image%2020261115171000.png)

Hydra also reveals that the user `SHaylett` uses his username as a password. SSH is used to gain access to the target system as `SHaylett`.
![](assets/stapler/Pasted%20image%2020261115171622.png)
## Privilege Escalation
The `find` command is used to search all writable files on the target. The output shows the `/usr/local/sbin/cron-logrorate.sh` script.
![](assets/stapler/Pasted%20image%2020261115171228.png)

Inspection of `/etc/cron.d/logrotate` reveals that the script is automatically executed by root every 5 minutes.
![](assets/stapler/Pasted%20image%2020261115171317.png)

A Bash command copying `/bin/bash` to `/tmp/rootbash` and setting the SUID bit on the copied binary is appended to the script.
![](assets/stapler/Pasted%20image%2020261115171444.png)

Shortly after, the `rootbash` binary is found in `/tmp` with the SUID bit set.
![](assets/stapler/Pasted%20image%2020261115171516.png)

Executing the binary with the `-p` flag spawns a root shell.
![](assets/stapler/Pasted%20image%2020261115171537.png)
## Finding & Remediation
### Anonymous FTP Access
#### Description
The FTP service allows anonymous authentication and provides access to a file containing potential usernames. The FTP banner also exposes a potential username.
#### Impact
An unauthenticated attacker can retrieve information about potential system users and use it to perform targeted password attacks against exposed authentication services.
#### Remediation
Disable anonymous FTP access unless explicitly required. Restrict access to internal files and remove unnecessary information from service banners.
### Unauthenticated SMB Enumeration
#### Description
The SMB service allows null authentication, exposing information about system users and available shares without requiring valid credentials.
#### Impact
An unauthenticated attacker can enumerate system usernames and accessible shares. The disclosed usernames can be used to perform targeted password attacks against services such as FTP and SSH.
#### Remediation
Disable unauthenticated SMB enumeration and restrict access to shares and user information to authorized accounts. Review the Samba configuration and enforce appropriate authentication and access controls.
### Use of Weak Credentials
#### Description
The `elly` account uses its username in reverse order as its password, while the `SHaylett` account uses its username as its password. Both passwords are easily predictable.
#### Impact
An attacker can guess the passwords through targeted password attacks and gain unauthorized access to the affected accounts. In this case, the recovered credentials allowed FTP authentication as `elly` and SSH authentication as `SHaylett`.
#### Remediation
Require strong, unique passwords and prevent users from selecting passwords based on their usernames. Rotate the compromised passwords and implement rate limiting on authentication services.
### Misconfigured Cron Job
#### Description
The `/usr/local/sbin/cron-logrorate.sh` script is automatically executed by root every five minutes through a cron job configured in `/etc/cron.d/logrotate`. The script can be modified by an unprivileged user.
#### Impact
An attacker with write access to the script can modify its contents to execute arbitrary commands with root privileges during the next scheduled execution. In this case, exploitation resulted in a SUID-enabled Bash binary and a root shell.
#### Remediation
Ensure that the script is owned by `root` and remove write permissions for unprivileged users. Verify that its parent directories cannot be modified by unauthorized users and audit other scripts executed through privileged cron jobs.

