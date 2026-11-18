# Workaholic (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show FTP running on port 21, SSH running on port 22 and an Nginx web server hosting WordPress on port 80.
![](assets/workaholic/Pasted%20image%2020261115211754.png)

The web application doesn't load correctly and initiates redirections to the `workaholic.offsec` host, which is added to `/etc/hosts`.
![](assets/workaholic/Pasted%20image%2020261115211855.png)

The WordPress application is then accessed when browsing to `http://workaholic.offsec`.
![](assets/workaholic/Pasted%20image%2020261115211950.png)

User enumeration with WPScan shows three valid users: `admin`, `ted` and `charlie`.
![](assets/workaholic/Pasted%20image%2020261115212107.png)

WPScan also suggests that the application is vulnerable to an unauthenticated SQL Injection vulnerability (CVE-2024-9796).
![](assets/workaholic/Pasted%20image%2020261115212507.png)

The WPScan website provides a proof of concept `curl` command to exploit this vulnerability and obtain users' credentials.
![](assets/workaholic/Pasted%20image%2020261115212528.png)

The URL is accessed, revealing the hashed passwords for the identified users.
![](assets/workaholic/Pasted%20image%2020261115212658.png)

John the Ripper successfully cracks the password hashes for both `charlie` and `ted`.
![](assets/workaholic/Pasted%20image%2020261115212801.png)
![](assets/workaholic/Pasted%20image%2020261115212748.png)

Neither set of credentials is valid for SSH authentication. However, the `ted` user's credentials can be used for the FTP service.
![](assets/workaholic/Pasted%20image%2020261115212823.png)

After turning off FTP passive mode, the directory is successfully listed. The FTP service seems to run from the WordPress web application's root directory.
![](assets/workaholic/Pasted%20image%2020261115212926.png)
## Initial Access
The `wp-config.php` file is downloaded from FTP and inspected, revealing the database password for the `wpadmin` user.
![](assets/workaholic/Pasted%20image%2020261115213044.png)

Since Nmap did not identify a running MySQL server, the password is sprayed against all three of the previously identified users. The output from NetExec confirms that the password is valid for SSH authentication as the user `charlie`.
![](assets/workaholic/Pasted%20image%2020261115213217.png)

Direct access to the target is obtained through the SSH service.
![](assets/workaholic/Pasted%20image%2020261115213455.png)
## Privilege Escalation
Searching for files with the SUID or SGID bit set using the `find` command reveals that `wp-monitor` in the WordPress web application's directory is owned by `root` and has the SUID bit set.
![](assets/workaholic/Pasted%20image%2020261115213530.png)

Using `strings` on the file shows that it is attempting to use the `init_plugin` function from `/home/ted/.lib/libsecurity.so`.
![](assets/workaholic/Pasted%20image%2020261115213634.png)

The `ted` user's home directory is writable by all users on `workaholic`.
![](assets/workaholic/Pasted%20image%2020261115213924.png)

The `.lib` directory is created since it does not exist on the target. The following code is used to create a malicious `libsecurity.so` file that will set the SUID bit on `/bin/bash`.
![](assets/workaholic/Pasted%20image%2020261115215018.png)

The file is compiled using `gcc`.
![](assets/workaholic/Pasted%20image%2020261115215345.png)

The `wp-monitor` binary is then executed.
![](assets/workaholic/Pasted%20image%2020261115215409.png)

This results in the `/bin/bash` binary having the SUID bit set.
![](assets/workaholic/Pasted%20image%2020261115215627.png)

A root shell is spawned by executing `/bin/bash -p`.
![](assets/workaholic/Pasted%20image%2020261115215652.png)

## Finding & Remediation
### Unauthenticated SQL Injection (CVE-2024-9796)
#### Description
The WP-Advanced-Search WordPress plugin is vulnerable to SQL injection due to insufficient validation of user-controlled input before it is used in SQL queries. The vulnerability does not require authentication.
#### Impact
An unauthenticated attacker can access sensitive information stored in the application's database. In this case, exploitation of the vulnerability exposed WordPress user password hashes.
#### Remediation
Upgrade the WP-Advanced-Search plugin to version 3.3.9.2 or later. Ensure that database queries use parameterized statements and properly validate user-controlled input. Reset the passwords of affected users.
### Password Reuse
#### Description
Identical passwords are used across separate accounts and authentication services. The `ted` user's WordPress password is reused for FTP authentication, while the WordPress database password for `wpadmin` is also used by the local `charlie` account.
#### Impact
Compromised credentials can provide unauthorized access to additional accounts and services. In this case, password reuse resulted in unauthorized FTP and SSH access to the target.
#### Remediation
Enforce unique passwords across application, database and operating system accounts. Rotate all compromised passwords and ensure that service account credentials are not reused for personal accounts.
### Insecure Shared Library Loading in SUID Binary
#### Description
The root-owned `wp-monitor` binary has the SUID bit set and loads a shared library from a directory that is writable by unprivileged users. The binary does not sufficiently verify the integrity or ownership of the library it loads.
#### Impact
An unprivileged user can compromise the integrity of the shared library loading process, resulting in arbitrary code execution with root privileges. In this case, exploitation resulted in full root access to the target.
#### Remediation
Remove the SUID bit from the `wp-monitor` binary unless elevated privileges are explicitly required. Ensure that shared libraries are loaded exclusively from trusted, root-controlled directories. Restrict write permissions on the libraries and their parent directories to authorized administrative users.
