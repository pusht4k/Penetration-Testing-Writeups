# Scrutiny (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show SSH running on port 22, SMTP running on port 25 and an Apache web server running on port 80.
![](assets/scrutiny/Pasted%20image%2020261111221534.png)

The web application contains a `Login` button at the top right corner. The button initiates a redirection to `http://teams.onlyrands.com`.
![](assets/scrutiny/Pasted%20image%2020261111221621.png)

The hostname is added to `/etc/hosts`, resolving it to the target's IP address. When accessed, the web application redirects to the TeamCity login page. The TeamCity installation is version 2023.05.4 as shown below the `Log in` button.
![](assets/scrutiny/Pasted%20image%2020261111221704.png)

A Google search leads to CVE-2024-27198, an authentication bypass vulnerability affecting TeamCity installations prior to version 2023.11.4, with a public proof of concept available on [yoryio's GitHub](https://github.com/yoryio/CVE-2024-27198).
The exploit is successful, creating a new user with both the username and the password set to `pusht4k`.
![](assets/scrutiny/Pasted%20image%2020261111222217.png)
## Initial Access
The TeamCity dashboard is accessed using the newly created user's credentials.
![](assets/scrutiny/Pasted%20image%2020261111222355.png)

A private SSH key is found under `Projects > Freelancers > Marco Tillman > Change Log`. The username of marco is also revealed as `marcot`.
![](assets/scrutiny/Pasted%20image%2020261111222845.png)

The key is directly accessed through the web application, then saved to a file.
![](assets/scrutiny/Pasted%20image%2020261111222518.png)

John the Ripper is used to extract and crack the key's passphrase.
![](assets/scrutiny/Pasted%20image%2020261111222627.png)

The target is then directly accessed through SSH.
![](assets/scrutiny/Pasted%20image%2020261111222654.png)
## Privilege Escalation
Since SMTP is running on the `onlyrands` host, the `/var/spool/mail` directory is checked for stored emails, revealing a stored email for the `marcot` user.
![](assets/scrutiny/Pasted%20image%2020261111222955.png)

The email contains a message from `matthewa`, revealing the password for the user's account. The email also mentions Dach who is supposedly another worker, as well as a parting gift that only Marco can access.
![](assets/scrutiny/Pasted%20image%2020261111223038.png)

The password is used with `su` to switch to `matthewa`'s account. A hidden file is then found in `matthewa`'s home directory.
![](assets/scrutiny/Pasted%20image%2020261111223304.png)

The hidden file contains the plaintext password for Dach's account.
![](assets/scrutiny/Pasted%20image%2020261111223339.png)

Inspection of `/etc/passwd` reveals an administrator named Brian Dach with the user name `briand`.
![](assets/scrutiny/Pasted%20image%2020261111223448.png)

The `su` command is then used to switch to the `briand` account.
![](assets/scrutiny/Pasted%20image%2020261111223645.png)

The `sudo -l` command reveals that `briand` can execute `/usr/bin/systemctl status teamcity-server.service` as root with no password requirement.
![](assets/scrutiny/Pasted%20image%2020261111223751.png)

When executing the sudo command the service status is displayed in a LESS pager.
![](assets/scrutiny/Pasted%20image%2020261111223810.png)

Since the command was executed with sudo, the less pager is also running as `root`. A root shell is spawned using the less pager `!/bin/bash` command.
![](assets/scrutiny/Pasted%20image%2020261111223913.png)

## Finding & Remediation
### TeamCity Authentication Bypass (CVE-2024-27198)
#### Description
The TeamCity installation is running version 2023.05.4, which is vulnerable to CVE-2024-27198. This vulnerability allows unauthenticated attackers to bypass authentication checks and perform administrative actions through specially crafted HTTP requests.
#### Impact
An unauthenticated attacker can exploit the vulnerability to create a new administrative user and gain unauthorized access to the TeamCity application.
#### Remediation
Upgrade TeamCity to version 2023.11.4 or later, which addresses CVE-2024-27198. If upgrading is not immediately possible, apply the official security patch plugin provided by JetBrains.
### Sensitive Information Disclosure
#### Description
A private SSH key that was previously removed from a TeamCity change log remains accessible through the web application. The key is protected by a passphrase but can still be retrieved by users with access to the affected resource.
#### Impact
An attacker who gains access to the private key and obtains or cracks its passphrase can use it to authenticate to the associated SSH account. In this case, the exposed key allowed SSH access as `marcot`.
#### Remediation
Remove exposed private keys from accessible application resources, including retained file versions or attachments. Revoke and replace compromised SSH keys, restrict access to sensitive project resources and ensure that deleted secrets cannot be retrieved through the application.
### Insecure Credential Storage
#### Description
User passwords are stored in plaintext within a local email and a hidden file in a user's home directory. These files expose credentials belonging to other system accounts, including an administrator.
#### Impact
An attacker with access to the exposed files can obtain credentials for additional system accounts. In this case, the disclosed passwords allowed access to the `matthewa` and `briand` accounts, enabling further lateral movement and privilege escalation.
#### Remediation
Remove plaintext passwords from emails and files accessible to unauthorized users. Rotate all exposed credentials, enforce restrictive file permissions and use a secure password manager for storing and sharing credentials.
### Sudo Misconfiguration
#### Description
The `briand` account is permitted to execute `/usr/bin/systemctl status teamcity-server.service` with root privileges through sudo without requiring a password. The command launches a `less` pager without secure mode enabled, allowing shell commands to be executed with elevated privileges.
#### Impact
An attacker with access to the `briand` account can exploit the unrestricted pager to execute arbitrary commands as `root`, resulting in complete system compromise. Since the sudo command does not require a password, privilege escalation is possible even if the attacker gains access to the account without knowing its password.
#### Remediation
Disable the interactive pager when executing `systemctl` with elevated privileges by using the `--no-pager` option in the permitted sudo command. Alternatively, explicitly enforce secure pager mode using `SYSTEMD_PAGERSECURE=1`. Review sudo rules to prevent privileged commands from launching unrestricted interactive programs and avoid granting passwordless sudo access unless strictly necessary.

