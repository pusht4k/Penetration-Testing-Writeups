# SpiderSociety (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results suggest the target is a Linux machine, with SSH running on port 22, an Apache web server running on port 80 and FTP running on port 2121.
![](assets/spidersociety/Pasted%20image%2020261115180122.png)

The web application's home page is seemingly empty. The `Login` button redirects to the 404 page.
![](assets/spidersociety/Pasted%20image%2020261115180431.png)

A directory brute-force scan with `gobuster` reveals a `/libspider` endpoint. The endpoint is a login page for the Spider Society Control Panel.
![](assets/spidersociety/Pasted%20image%2020261115180522.png)

An attempt to authenticate with the credentials `admin:admin` is successful and redirects to the control panel.
![](assets/spidersociety/Pasted%20image%2020261115180604.png)

When clicking on the `Communications` button, a message pops up containing credentials for a backup user.
![](assets/spidersociety/Pasted%20image%2020261115180811.png)
## Initial Access
These credentials are used to authenticate through FTP.
The files in the directory suggest that the service is running from the web root.
![](assets/spidersociety/Pasted%20image%2020261115180934.png)

This is further confirmed when the `simple.py` file found in the FTP directory is accessed through the browser.
![](assets/spidersociety/Pasted%20image%2020261115181041.png)

Further testing reveals that file uploads are allowed through FTP to the web root directory, and the files are accessible. The PHP reverse shell script from PentestMonkey is then prepared in order to get a reverse shell.
![](assets/spidersociety/Pasted%20image%2020261115181321.png)

Additionally, an `.htaccess` file is set up to enable the execution of PHP files.
![](assets/spidersociety/Pasted%20image%2020261115181354.png)

Both files are then uploaded through the FTP service and `chmod` is used to grant the PHP file execution permissions.
![](assets/spidersociety/Pasted%20image%2020261115181452.png)

The file is accessed through the browser, resulting in a reverse shell as `www-data`.
![](assets/spidersociety/Pasted%20image%2020261115181541.png)
## Privilege Escalation
Inspection of the `fetch-credentials.php` file in the `libspider` directory reveals that credentials are extracted from a hidden file in the current working directory.
![](assets/spidersociety/Pasted%20image%2020261115181635.png)

The hidden file contains database credentials for the `spidey` user.
![](assets/spidersociety/Pasted%20image%2020261115181729.png)

The `/home` directory contains a `spidey` directory, indicating that this is also a local user on the target.
![](assets/spidersociety/Pasted%20image%2020261115181800.png)

The password is then used to authenticate through SSH as the `spidey` user.
![](assets/spidersociety/Pasted%20image%2020261115181829.png)

Enumeration of sudo permissions reveals that `spidey` can execute `/bin/systemctl` to reload the systemd configuration using `daemon-reload` and to restart `spiderbackup.service`.
![](assets/spidersociety/Pasted%20image%2020261115181928.png)

The `spiderbackup.service` file is writable by the compromised user.
![](assets/spidersociety/Pasted%20image%2020261115182006.png)

The service file executes `/usr/local/bin/spiderbackup.sh` when the service starts.
![](assets/spidersociety/Pasted%20image%2020261115182037.png)

A malicious `spiderbackup.sh` script that sets the SUID bit on `/bin/bash` is created in the `spidey` user's home directory.
![](assets/spidersociety/Pasted%20image%2020261115182117.png)

The `spiderbackup.service` file is then modified to point to `/home/spidey/spiderbackup.sh`.
![](assets/spidersociety/Pasted%20image%2020261115182218.png)

The sudo commands are used to reload the systemd configuration and then restart `spiderbackup.service`.
![](assets/spidersociety/Pasted%20image%2020261115182329.png)

The bash binary is then found with the SUID bit set and executed using the `-p` parameter to maintain root privileges.
![](assets/spidersociety/Pasted%20image%2020261115182404.png)
## Finding & Remediation
### Use of Weak Credentials
#### Description
The Spider Society Control Panel uses the easily guessable credentials `admin:admin` for its administrative account.
#### Impact
An attacker can gain unauthorized administrative access to the control panel by guessing the credentials. In this case, access to the control panel exposed credentials for the FTP backup account.
#### Remediation
Replace the weak administrative password with a strong, unique password. Enforce a strong password policy and implement rate limiting on authentication attempts.
### Insecure FTP Configuration
#### Description
The FTP backup account has write access to the web root directory despite file uploads being restricted within the `libspider` directory. This inconsistency suggests that write access to the web root may be unintended. Additionally, the Apache configuration permits uploaded `.htaccess` files to modify PHP execution settings within that directory.
#### Impact
An attacker with access to the FTP account can upload malicious PHP scripts and modify the web server's configuration to enable their execution. In this case, exploitation resulted in a reverse shell as `www-data`.
#### Remediation
Restrict the FTP backup account to a dedicated directory outside the web root and grant only the permissions required for backup operations. Review directory permissions to ensure that upload restrictions are consistently enforced. Disable unnecessary `.htaccess` overrides using `AllowOverride None` and prevent the execution of scripts in directories where file uploads are permitted.
### Password Reuse
#### Description
The `spidey` database account and the local `spidey` account use the same password across separate authentication mechanisms.
#### Impact
An attacker who obtains the database credentials can reuse the password to authenticate as the local `spidey` user. In this case, password reuse enabled SSH access to the target using credentials retrieved from a database configuration file.
#### Remediation
Use strong, unique passwords for database and operating system accounts. Rotate the reused password on both accounts and avoid sharing credentials between services.
### Sudo Misconfiguration
#### Description
The `spidey` account is permitted to reload the systemd configuration and restart `spiderbackup.service` with root privileges through sudo. However, the service unit file is writable by the `spidey` user, allowing its execution directives to be modified.
#### Impact
An attacker with access to the `spidey` account can modify the service unit file to execute arbitrary commands with root privileges and apply the changes using the permitted sudo commands. In this case, exploitation resulted in a SUID-enabled Bash binary and a root shell.
#### Remediation
Remove unnecessary sudo permissions for systemctl operations. Ensure that the service unit file, its associated scripts and their parent directories are owned and controlled by trusted administrative users. Remove write permissions for unprivileged users from all files involved in privileged service execution.