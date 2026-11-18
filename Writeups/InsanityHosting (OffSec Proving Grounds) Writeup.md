# InsanityHosting (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results indicate that the target is a Linux host, with FTP running on port 21, SSH running on port 22 and an Apache web server running on port 80.
```
┌──(root㉿shushan)-[/home/shushan/OffSec/Labs/InsanityHosting]
└─# nmap -p- -A -T4 192.168.135.124
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-26 20:46 +0300
Nmap scan report for 192.168.135.124
Host is up (0.089s latency).
Not shown: 65381 filtered tcp ports (no-response), 151 filtered tcp ports (host-prohibited)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.45.207
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.2 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: ERROR
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 85:46:41:06:da:83:04:01:b0:e4:1f:9b:7e:8b:31:9f (RSA)
|   256 e4:9c:b1:f2:44:f1:f0:4b:c3:80:93:a9:5d:96:98:d3 (ECDSA)
|_  256 65:cf:b4:af:ad:86:56:ef:ae:8b:bf:f2:f0:d9:be:10 (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/7.2.33)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Insanity - UK and European Servers
|_http-server-header: Apache/2.4.6 (CentOS) PHP/7.2.33
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running (JUST GUESSING): Linux 3.X|4.X|2.6.X|5.X (97%), MikroTik RouterOS 7.X (91%)
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:5 cpe:/o:linux:linux_kernel:6 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
Aggressive OS guesses: Linux 3.10 - 4.11 (97%), Linux 3.2 - 4.14 (97%), Linux 3.13 - 4.4 (91%), Linux 3.8 - 3.16 (91%), Linux 2.6.32 - 3.13 (91%), Linux 3.4 - 3.10 (91%), Linux 4.15 (91%), Linux 4.15 - 5.19 (91%), Linux 5.0 - 5.14 (91%), Linux 5.14 - 6.8 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 4 hops
Service Info: OS: Unix
<SNIP>
```

A `gobuster` scan finds the `news`, `monitoring`, `phpmyadmin` and `webmail` endpoints.
![](assets/insanityhosting/Pasted%20image%2020261110150807.png)

The `webmail` endpoint is an installation of SquirrelMail version 1.4.22. Attempts at guessing credentials and SQL Injection authentication bypass mechanisms fail.
![](assets/insanityhosting/Pasted%20image%2020261110150853.png)

The `news` endpoint displays a blog post that mentions Otis, a team member, as well as the `insanityhosting.vm` hostname.
![](assets/insanityhosting/Pasted%20image%2020261110151007.png)

A brute-force attempt against the SquirrelMail application is made using `otis` as the username. The attempt is successful, revealing that the user's password is `123456`.
![](assets/insanityhosting/Pasted%20image%2020261110151128.png)

The `monitoring` endpoint features another login page, with the credentials of `otis` reused to gain access to the dashboard. The application allows adding IP addresses and tests whether the web server is up, then sends an email containing the results, which can be accessed through the SquirrelMail application.
![](assets/insanityhosting/Pasted%20image%2020261110151229.png)

The `OR 1=1 `SQL clause is added as a server, which leads the application to send an email with all previous checks for localhost, confirming that the target is vulnerable to SQL Injection.
![](assets/insanityhosting/Pasted%20image%2020261110151451.png)
## Initial Access
An `UNION SELECT` statement is then added to enumerate the different databases hosted on the server, exposing the default `mysql` database.
![](assets/insanityhosting/Pasted%20image%2020261110151831.png)

The following query is then used to retrieve all table and column names from the `mysql` database
```SQL
" UNION SELECT null, table_name, column_name, table_schema from information_schema.columns where table_schema='mysql' -- -
```

The results show a `user` table containing the `User`, `Password` and `authentication_string` columns. 
![](assets/insanityhosting/Pasted%20image%2020261110152112.png)

A subsequent query retrieves information from the `user` table, revealing the user `elliot` with the user's password hash.
![](assets/insanityhosting/Pasted%20image%2020261110152236.png)

CrackStation is used to crack the user's hash and recover the plaintext password: elliot123.
![](assets/insanityhosting/Pasted%20image%2020261110152345.png)

These credentials are reused to authenticate to the target through SSH as the user `elliot`.
![](assets/insanityhosting/Pasted%20image%2020261110152418.png)
## Privilege Escalation
`elliot`'s home directory contains a `.mozilla` directory.
![](assets/insanityhosting/Pasted%20image%2020261110152841.png)

A Mozilla Firefox profile is found in the directory containing the `key4.db` and `logins.json` files, indicating saved Firefox credentials.
![](assets/insanityhosting/Pasted%20image%2020261110152956.png)

A new `hosting` Firefox profile is created on the attacking machine. The `key4.db` and `logins.json` files are transferred to the root directory of the profile.
![](assets/insanityhosting/Pasted%20image%2020261110153047.png)

After launching the profile in a new browser and accessing the `Passwords` page, `root` credentials for a service running on port 10000 on the target are found.
![](assets/insanityhosting/Pasted%20image%2020261110153247.png)

The password is reused with `su` to switch to the `root` user.
![](assets/insanityhosting/Pasted%20image%2020261110153602.png)
## Finding & Remediation

### Weak Passwords
#### Description
Multiple user accounts were configured with weak and easily guessable passwords. The `otis` account used the password `123456`, which was discovered through a brute-force attack, while the `elliot` account used `elliot123`, which was recovered using CrackStation.
#### Impact
An attacker may guess or crack weak passwords to gain unauthorized access to user accounts and compromise additional services.
#### Remediation
Enforce a strong password policy that requires sufficiently long and unique passwords. Block commonly used and compromised passwords, and implement rate limiting to mitigate brute-force attacks.
### Password Reuse
#### Description
Multiple accounts reused the same credentials across different services:
- The `otis` account used the same password for SquirrelMail and the Monitoring application.
- The `elliot` account used the same password for MySQL and SSH authentication.
- The password stored in Firefox for a service running on `localhost:10000` was also used for the system's `root` account.
#### Impact
Compromising credentials for one service allows an attacker to access additional services or accounts. In this case, password reuse enabled SSH access as `elliot` and privilege escalation to `root`.
#### Remediation
Require unique passwords for separate services and accounts, particularly privileged accounts. Avoid reusing system account passwords for web applications or other services.
### SQL Injection
#### Description
The Monitoring application was vulnerable to SQL Injection through its server input field. User-supplied SQL statements were executed without proper sanitization or parameterization, allowing unauthorized queries against the database.
#### Impact
An attacker with access to the Monitoring application could enumerate databases, tables, and columns, and extract sensitive information, including MySQL user password hashes.
#### Remediation
Use parameterized queries or prepared statements to prevent user input from being interpreted as SQL code. Apply the principle of least privilege to database accounts to restrict access to sensitive system tables.

