# Potato (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan results show SSH running on port 22, an Apache web server running on port 80 and ProFTPD running on port 2112 with anonymous login enabled. Nmap also lists two files in the FTP root directory.
![](assets/potato/Pasted%20image%2020261112000034.png)

The `index.php.bak` contains a password variable with the `potato` value.
![](assets/potato/Pasted%20image%2020261112000316.png)

An `/admin` endpoint is found on the web server, featuring a login page. Login attempts with the `admin:potato` credentials fail.
![](assets/potato/Pasted%20image%2020261112000440.png)

A `gobuster` scan against the admin endpoint finds the `/logs` directory.
![](assets/potato/Pasted%20image%2020261112000540.png)

Directory listing is enabled and three log files are accessible.
![](assets/potato/Pasted%20image%2020261112000616.png)

The log files confirm that the password for the `admin` user has changed.
![](assets/potato/Pasted%20image%2020261112000808.png)
## Initial Access
Further inspection of `index.php.bak` reveals that the application uses loose comparison (`==`) instead of strict comparison (`===`) to compare the results of `strcmp()` against `0`. This allows an attacker to supply a null value, causing `strcmp()` to return `NULL`, which is loosely equal to `0` and bypasses the authentication mechanism.
![](assets/potato/Pasted%20image%2020261112000852.png)

The `password` parameter is set to NULL in Burp Suite, resulting in successful authentication and a redirection to the dashboard.
![](assets/potato/Pasted%20image%2020261112001132.png)

The dashboard uses the `page` parameter to display different pages.
![](assets/potato/Pasted%20image%2020261112001231.png)

When accessing the `Logs` directory, a POST request is made instead using an additional `file` parameter in the request's body.
![](assets/potato/Pasted%20image%2020261112001311.png)

The parameter is tested and confirmed vulnerable to local file inclusion. The `/etc/passwd` file's contents are retrieved, revealing the `webadmin` user's hashed password.
![](assets/potato/Pasted%20image%2020261112001414.png)

The password hash is cracked with John the Ripper, revealing the plaintext value: `dragon`.
![](assets/potato/Pasted%20image%2020261112001450.png)

The credentials are used to authenticate to the target through SSH as `webadmin` and retrieve the user flag from webadmin's home directory.
![](assets/potato/Pasted%20image%2020261112001522.png)
## Privilege Escalation
The `sudo -l` command reveals that `webadmin` can run `/bin/nice /notes/*` through sudo.
![](assets/potato/Pasted%20image%2020261112001607.png)

The `nice` command can be used to execute other programs, including `/bin/bash`. Although the sudo rule restricts execution to `/notes/*`, the wildcard allows directory traversal using `..` to reference the parent directory. By specifying `/notes/../bin/bash`, the path resolves to `/bin/bash`, allowing the command to spawn a root shell.
![](assets/potato/Pasted%20image%2020261112001715.png)

## Finding & Remediation
### Anonymous FTP Access
#### Description
The ProFTPD service allows anonymous authentication and provides access to sensitive files, including an `index.php.bak` backup file containing the web application's authentication logic and a hardcoded password.
#### Impact
An unauthenticated attacker can retrieve the backup file and inspect the application's source code, exposing sensitive information and revealing weaknesses in the authentication mechanism.
#### Remediation
Disable anonymous FTP access unless explicitly required. Restrict access to sensitive files and ensure that application source code, configuration files and backups are not accessible through publicly exposed services.
### Authentication Bypass
#### Description
The web application uses loose comparison (`==`) instead of strict comparison (`===`) when comparing the result of `strcmp()` against `0`. In the affected PHP environment, supplying an array instead of a string causes `strcmp()` to return `NULL`, which is considered equal to `0` during loose comparison.
#### Impact
An unauthenticated attacker can manipulate the login request to bypass password verification and gain unauthorized access to the administrative dashboard without valid credentials.
#### Remediation
Replace loose comparisons with strict comparisons (`===`) and validate input types before passing user-controlled values to `strcmp()`. Reject unexpected input types and update PHP to a supported version that enforces stricter type handling.
### Insecure Credential Storage
#### Description
The `webadmin` account's password hash is stored in the publicly readable `/etc/passwd` file instead of `/etc/shadow`. Additionally, the password is hashed using MD5-crypt (`$1$`), an outdated password hashing algorithm that provides insufficient resistance against modern password-cracking attacks.
#### Impact
An attacker with access to the system or the ability to read `/etc/passwd` through another vulnerability can retrieve the password hash and perform offline password cracking. In this case, the password was successfully recovered and used to authenticate as `webadmin` through SSH.
#### Remediation
Store password hashes in `/etc/shadow` with restrictive permissions and ensure that `/etc/passwd` contains only password placeholders. Replace MD5-crypt with a modern password hashing algorithm such as yescrypt and require affected users to change their passwords.
### Sudo Misconfiguration
#### Description
The `webadmin` account is permitted to execute `/bin/nice /notes/*` with root privileges through sudo. The `nice` utility can execute other programs, while the wildcard in the sudo rule allows directory traversal sequences such as `..`, enabling execution of binaries outside the intended `/notes` directory.
#### Impact
An attacker with access to the `webadmin` account can bypass the intended path restriction by specifying `/notes/../bin/bash`, causing `nice` to execute `/bin/bash` as `root`. This results in complete system compromise.
#### Remediation
Remove the unnecessary sudo permission or replace the wildcard-based rule with an explicit allowlist of required commands and arguments. Avoid granting elevated privileges to utilities that can execute arbitrary programs and ensure that sudo rules cannot be bypassed through directory traversal.
