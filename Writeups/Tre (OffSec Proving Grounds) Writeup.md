# Tre (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal SSH running on port 22, Apache running on port 80 and Nginx running on port 8082.
![](assets/tre/Pasted%20image%2020261115161959.png)

Both web applications display a static image on the home page.
![](assets/tre/Pasted%20image%2020261115162110.png)

A `gobuster` search reveals the `cms` and `mantisbt` endpoints on the web application running on port 80.
![](assets/tre/Pasted%20image%2020261115162223.png)

The `/mantisbt` endpoint features a login page for Mantis bug tracker. Attempts at guessing default credentials fail.
![](assets/tre/Pasted%20image%2020261115162322.png)

A second `gobuster` scan aimed at the `mantisbt` endpoint finds a `config` endpoint.
![](assets/tre/Pasted%20image%2020261115162416.png)

Directory listing is enabled at the `config` directory, revealing a TXT file.
![](assets/tre/Pasted%20image%2020261115162450.png)

The TXT file contains database credentials for `mantissuser`.
![](assets/tre/Pasted%20image%2020261115162537.png)
## Initial Access
The credentials are tested against the Mantis bug tracker application, but no further exploitation is achieved. Further enumeration of `php` files using gobuster reveals the `adminer.php` endpoint on the web root.
![](assets/tre/Pasted%20image%2020261115162630.png)

The database credentials are valid for adminer, providing access to the mantis database.
![](assets/tre/Pasted%20image%2020261115162708.png)

The `mantis_user_table` table contains credentials for users `administrator` and `tre`. The `realname` column seems to store the users' plaintext passwords.
![](assets/tre/Pasted%20image%2020261115162801.png)

The credentials for the `tre` user are valid and initial access is obtained through SSH authentication.
![](assets/tre/Pasted%20image%2020261115162937.png)
## Privilege Escalation
The `pspy` tool displays a `check-system` process executed automatically by UID=0 (`root`).
![](assets/tre/Pasted%20image%2020261115163014.png)

The `check-system` file is writable by all users on the target system.
![](assets/tre/Pasted%20image%2020261115163125.png)

The `check-system` script logs the service's start time to the systemd journal and enters an infinite loop, printing `Checking...` every second without performing any actual system checks.
![](assets/tre/Pasted%20image%2020261115163406.png)

Since `check-system` runs with elevated privileges, it can be overwritten with a command that will also be executed as `root`. The script is overwritten to set the SUID bit on `/bin/bash`.
![](assets/tre/Pasted%20image%2020261115163148.png)

After waiting for a while, the script has not been executed again, which could be due to it being stuck in an infinite loop. Further enumeration reveals that the user `tre` can run `/usr/bin/shutdown` through `sudo`.
![](assets/tre/Pasted%20image%2020261115163642.png)

Since the script runs automatically, restarting the machine should stop the infinite loop. The script should then execute properly. The target machine is restarted with the following command.
![](assets/tre/Pasted%20image%2020261115163730.png)

Shortly after, the `tre` host is up and the `/bin/bash` binary is found to have the SUID bit set.
![](assets/tre/Pasted%20image%2020261115163911.png)

The binary is executed with the `-p` parameter to obtain a root shell.
![](assets/tre/Pasted%20image%2020261115164024.png)

## Finding & Remediation
### Sensitive Information Disclosure
#### Description
Directory listing is enabled on the Mantis Bug Tracker configuration directory, exposing a text file containing plaintext database credentials. The file is accessible through the web application without authentication.
#### Impact
An unauthenticated attacker can retrieve the exposed database credentials and use them to authenticate through the Adminer interface, gaining unauthorized access to the Mantis database and its contents.
#### Remediation
Disable directory listing and prevent unauthorized access to configuration files. Store sensitive configuration files outside the web root, enforce restrictive file permissions and rotate the exposed database credentials.
### Insecure Credential Storage
#### Description
User passwords are stored in plaintext within the `realname` column of the `mantis_user_table` database table. Storing passwords in an unrelated database column does not provide any additional protection against unauthorized access.
#### Impact
An attacker who gains access to the database can retrieve user passwords without performing password cracking. In this case, the exposed password allowed the tester to authenticate to the underlying system through SSH as `tre`.
#### Remediation
Remove plaintext passwords from the database and store application passwords using a secure password hashing algorithm such as Argon2id or bcrypt. Rotate all exposed passwords and avoid password reuse between application and operating system accounts.
### Insecure Service Script Permissions
#### Description
The `/usr/bin/check-system` script is automatically executed with root privileges during system startup. The script grants write permissions to all users, allowing unprivileged users to modify its contents.
#### Impact
An attacker with access to a local user account can modify the script to execute arbitrary commands with root privileges upon the next system startup, resulting in complete system compromise.
#### Remediation
Ensure that the script is owned by `root` and remove write permissions for unauthorized users. Verify that the script's parent directories cannot be modified by unprivileged users and audit other scripts executed automatically with elevated privileges.
### Sudo Misconfiguration
#### Description
The `tre` account is permitted to execute `/usr/bin/shutdown` with root privileges through sudo without requiring a password.
#### Impact
An attacker who gains access to the `tre` account can shut down or restart the system without knowing the account's password, causing a denial of service. This permission can also be combined with the insecure service script permissions to trigger the execution of malicious code with root privileges.
#### Remediation
Remove the sudo permission for `/usr/bin/shutdown` unless it is required for legitimate administrative operations. If the permission is necessary, remove the `NOPASSWD` directive to require password authentication before allowing the user to shut down or restart the system.
