# Loly (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show only port 80 open on the target machine. The service is identified as Nginx 1.10.3, with the `http-title` indicating that the home page is the default Nginx page.
![](assets/loly/Pasted%20image%2020261111001009.png)

A `gobuster` directory brute-force scan finds the `wordpress` directory on the web server.
![](assets/loly/Pasted%20image%2020261111001153.png)

An attempt to access the `wp-admin` endpoint results in a redirection to the `loly.lc` hostname.
![](assets/loly/Pasted%20image%2020261111001248.png)

The hostname is added to `/etc/hosts` to resolve it to the target's IP address. The application is then accessed, with a blog written by `loly` displayed on the main page.
![](assets/loly/Pasted%20image%2020261111001513.png)

User enumeration in `wp-login` reveals that `loly` is a valid username.
![](assets/loly/Pasted%20image%2020261111001419.png)

A brute-force attack using WPScan reveals the user's password: `fernando`.
```
└─# wpscan --url http://loly.lc/wordpress -U 'loly' -P /usr/share/wordlists/rockyou.txt
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.1.0
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://loly.lc/wordpress/ [192.168.215.121]
[+] Started: Sat Aug 29 21:50:32 2026
[+] Command Line: wpscan --url http://loly.lc/wordpress -U loly -P /usr/share/wordlists/rockyou.txt
[+] Hostname: shushan
<SNIP>

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - loly / fernando                                                                                                                                                    
Trying loly / corazon Time: 00:00:06 <=====================================================================================================> (175 / 175) 100.00% Time: 00:00:06

[!] Valid Combinations Found:
 | Username: loly, Password: fernando
```
## Initial Access
The credentials are then used to access the dashboard. The AdRotate WordPress plugin is installed on the web application. The plugin is identified as version 5.8.6.2.
![](assets/loly/Pasted%20image%2020261111001745.png)

A Google search leads to a public [GitHub repository by jephk9](https://github.com/jephk9/oscp-jewels/blob/main/services/wordpress-plugin-exploits.md) documenting an exploit in this version of AdRotate, allowing uploads of ZIP archives as image banners. The archives are then extracted automatically and saved to the `/banner` directory.
![](assets/loly/Pasted%20image%2020261111001857.png)

The `php-reverse-shell.php` script from PentestMonkey is prepared, pointing to port 443 on the attacking machine.
![](assets/loly/Pasted%20image%2020261111002145.png)

The file name is changed to `web.php`. The file is then stored in a ZIP archive.
![](assets/loly/Pasted%20image%2020261111002218.png)

The archive is uploaded successfully, and the web application returns the `Banner image saved` message.
![](assets/loly/Pasted%20image%2020261111002258.png)

Although the GitHub documentation indicates that banners are saved in the `/banner` directory, the dashboard shows it as empty. Instead, the file is found at `http://loly.lc/wordpress/wp-content/web.php`. Accessing this URL executes the file, resulting in a reverse shell as the `www-data` user.
![](assets/loly/Pasted%20image%2020261111002432.png)
## Privilege Escalation
The `wp-config.php` file in the WordPress directory contains database credentials.
![](assets/loly/Pasted%20image%2020261111002542.png)

The password is reused to obtain access to the `loly` user using the `su` command.
![](assets/loly/Pasted%20image%2020261111002617.png)

The `uname -a` and `cat /etc/os-release` commands show that the target is running Ubuntu 16.04.1 LTS (Xenial Xerus), with kernel version `4.4.0-31-generic`.
![](assets/loly/Pasted%20image%2020261111002746.png)

Searchsploit is used to find a Local Privilege Escalation vulnerability in this version (CVE-2017-16995).
![](assets/loly/Pasted%20image%2020261111003037.png)

The provided proof of concept was tested on Linux kernel `4.4.0-31-generic`, which is the exact kernel version of the target system.
![](assets/loly/Pasted%20image%2020261111003157.png)

The exploit is transferred to the target and compiled with `gcc` as specified in the proof of concept.
![](assets/loly/Pasted%20image%2020261111003302.png)

The compiled binary is then executed, successfully obtaining a shell as the `root` user.
![](assets/loly/Pasted%20image%2020261111003339.png)

## Finding & Remediation
### Use of Weak Passwords
#### Description
The WordPress account `loly` uses a weak password (`fernando`) that can be identified through a dictionary-based brute-force attack using WPScan.
#### Impact
An unauthenticated attacker can obtain access to the WordPress dashboard by guessing the account's password.
#### Remediation
Enforce strong, unique passwords for WordPress accounts and implement login rate limiting to mitigate brute-force attacks.
### Unrestricted File Upload - AdRotate Plugin
#### Description
The WordPress application uses AdRotate version 5.8.6.2, which allows authenticated users to upload ZIP archives containing PHP files as image banners. The uploaded files are automatically extracted into a web-accessible directory.
#### Impact
An attacker with access to the WordPress dashboard can upload and execute a malicious PHP file, obtaining a reverse shell as the `www-data` user.
#### Remediation
Update the AdRotate plugin to a maintained version that addresses arbitrary file upload vulnerabilities. Restrict banner uploads to approved image formats, validate file contents and archive entries, and prevent PHP execution in upload directories.
### Password Reuse
#### Description
The database password stored in the WordPress `wp-config.php` file is reused as the operating system password for the `loly` user.
#### Impact
An attacker with access to the WordPress configuration file can reuse the exposed database password to authenticate as the `loly` user, gaining access to that account.
#### Remediation
Use unique passwords for database and operating system accounts. Rotate the affected credentials and ensure that application credentials are not reused for system authentication.
### CVE-2017-16995 - Linux Kernel Privilege Escalation
#### Description
The target runs Ubuntu 16.04.1 LTS with Linux kernel `4.4.0-31-generic`, which is vulnerable to CVE-2017-16995. The vulnerability exists in the Berkeley Packet Filter (BPF) implementation and allows local privilege escalation through improper sign extension.
#### Impact
An attacker with local access to the system can exploit this vulnerability to obtain a shell with `root` privileges, resulting in complete system compromise.
#### Remediation
Update the Ubuntu 16.04 generic kernel to at least `4.4.0-119-generic` (package version `4.4.0-119.143`), which includes the fix for CVE-2017-16995, and reboot the system to activate the patched kernel. Since Ubuntu 16.04 is an outdated release, migrate to a currently supported Ubuntu LTS version to receive ongoing security updates.