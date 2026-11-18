# Snookums (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan shows a large number of open ports on the target, with the most notable ones being FTP running on port 21, SSH running on port 22, an Apache web server running on port 80 and SMB running on port 445.
![](assets/snookums/Pasted%20image%2020261115183749.png)

The web application's home page reveals that Simple PHP Photo Gallery v0.8 is in use.
![](assets/snookums/Pasted%20image%2020261115183913.png)

A Google search leads to a remote file inclusion exploit for Simple PHP Gallery 0.7 on Exploit-DB.
![](assets/snookums/Pasted%20image%2020261115184030.png)

While the application's version does not match the exploited version, this proof of concept is tested by setting the value of the `img` parameter to a non-existent file on the attacking machine.
![](assets/snookums/Pasted%20image%2020261115184151.png)

A GET request is then observed through a simple Python web server running on the attacking machine.
![](assets/snookums/Pasted%20image%2020261115184303.png)
## Initial Access
The reverse shell PHP script from PentestMonkey is set up in the same directory as the Python web server, pointing to port 3306 on the attacking machine. Port 3306 is commonly used for MySQL. Using this port for a reverse shell may bypass firewall restrictions.
![](assets/snookums/Pasted%20image%2020261115184456.png)

The `img` parameter is then set to point to the reverse shell script.
![](assets/snookums/Pasted%20image%2020261115184703.png)

The file is successfully executed on the target machine, resulting in a reverse shell as the `apache` user.
![](assets/snookums/Pasted%20image%2020261115184747.png)
## Privilege Escalation
Database credentials for the `root` user are found in `/var/www/html/db.php`.
![](assets/snookums/Pasted%20image%2020261115184911.png)

These credentials are used to authenticate to the local MySQL server. The `show databases;` SQL statement reveals the `SimplePHPGal` database.
![](assets/snookums/Pasted%20image%2020261115184947.png)

A `users` table is found in the database, storing credentials for `josh`, `michael` and `serena`. The passwords are seemingly encoded in Base64.
![](assets/snookums/Pasted%20image%2020261115185042.png)

Further testing reveals that the passwords are double-encoded in Base64. Using `base64 -d` twice successfully decodes the passwords, revealing their plaintext values.
![](assets/snookums/Pasted%20image%2020261115185129.png)

The `michael` user's password is used for SSH authentication.
![](assets/snookums/Pasted%20image%2020261115185302.png)

Enumeration of the `/etc/passwd` file reveals that `michael` is the owner of the file.
This can directly lead to privilege escalation by appending a new user with a controlled password and a UID of 0 (root).
![](assets/snookums/Pasted%20image%2020261115185537.png)

A password hash is then generated using `openssl`.
![](assets/snookums/Pasted%20image%2020261115185659.png)

A new `fakeroot` user is appended to `/etc/passwd` using `echo`.
![](assets/snookums/Pasted%20image%2020261115185756.png)

The `su` command is used to switch to the `fakeroot` user. The `id` command confirms that the UID is set to 0 (root).
![](assets/snookums/Pasted%20image%2020261115185835.png)

## Finding & Remediation
### Simple PHP Photo Gallery Remote File Inclusion (CVE-2022-40089)
#### Description
Simple PHP Photo Gallery is vulnerable to a remote file inclusion vulnerability due to insufficient validation of the `img` parameter. Although the publicly available exploit targets version 0.7, the same vulnerability was confirmed to affect version 0.8.
#### Impact
An unauthenticated attacker can include and execute remotely hosted PHP scripts on the target server. In this case, exploitation of the vulnerability resulted in a reverse shell as the `apache` user.
#### Remediation
Replace the vulnerable application with a maintained alternative or apply a security patch that addresses the vulnerability. Restrict file inclusion to explicitly allowed local files and disable remote file inclusion by setting `allow_url_include` to `Off` in the PHP configuration.
### Insecure Credential Storage
#### Description
User passwords are stored in the application's database using double Base64 encoding instead of a secure password hashing algorithm. Base64 is a reversible encoding mechanism that provides no cryptographic protection.
#### Impact
An attacker who gains access to the database can immediately recover the plaintext passwords without performing password cracking. In this case, the recovered password for `michael` allowed SSH authentication to the target.
#### Remediation
Store passwords using a secure password hashing algorithm such as Argon2id or bcrypt with unique salts. Reset all exposed passwords and enforce unique credentials across application and operating system accounts. Additionally, replace the MySQL `root` account used by the application with a dedicated database account that has only the permissions required for application functionality.
### Insecure File Permissions
#### Description
The `/etc/passwd` file is owned by the unprivileged `michael` user rather than `root`, allowing the user to modify its contents.
#### Impact
An attacker with access to the `michael` account can modify existing user entries or create new accounts with arbitrary user IDs. In this case, a new account with a controlled password and a UID of 0 was created, resulting in full root access to the target.
#### Remediation
Restore ownership of `/etc/passwd` to `root:root` and set its permissions to `644`. Ensure that only root can modify the file and audit other sensitive system files for incorrect ownership or excessive write permissions. Review existing user entries and remove any unauthorized accounts.