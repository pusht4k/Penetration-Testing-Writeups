# Jordak (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal that SSH is running on port 22 on the target and an Apache web server is running on port 80.
![](assets/jordak/Pasted%20image%2020261110154931.png)

The `docs` directory is accessible with directory listing enabled, providing access to multiple files.
![](assets/jordak/Pasted%20image%2020261110155036.png)

The `license.txt` file reveals an installation of the CodeIgniter application.
![](assets/jordak/Pasted%20image%2020261110155132.png)

The `install/` directory contains a `README` file for the installation of `Jorani`. The file includes the default credentials for the application, with `bbalet` serving as both the username and the password.
![](assets/jordak/Pasted%20image%2020261110155231.png)
## Initial Access
The `/jorani` endpoint leads to a login page. The default credentials are used to successfully authenticate to the web application and access the dashboard. 
![](assets/jordak/Pasted%20image%2020261110155425.png)

The service is identified as version v1.0.0 at the bottom of the page.
![](assets/jordak/Pasted%20image%2020261110155649.png)

A Google search leads to a publicly available proof of concept script on [samipmainali's GitHub](https://github.com/samipmainali/Jorani-Reverse-Shell-v1.0.0). 
The script takes advantage of an unrestricted file upload in addition to a path traversal vulnerability (CVE-2023-26469) to establish a reverse shell.
![](assets/jordak/Pasted%20image%2020261110155551.png)

The exploit is executed, pointing the reverse shell to port 443 on the attacking machine, which returns a successful output.
![](assets/jordak/Pasted%20image%2020261110160028.png)

A reverse shell is established, running as the user `jordak`.
![](assets/jordak/Pasted%20image%2020261110160141.png)
## Privilege Escalation
The `sudo -l` command reveals that `jordak` can either execute any command using `sudo` with a password or execute `/usr/bin/env` without one.
![](assets/jordak/Pasted%20image%2020261110160243.png)

Since the user's password is not known, the `sudo /usr/bin/env` command is abused to execute `/bin/bash`, resulting in a `root` shell.
![](assets/jordak/Pasted%20image%2020261110160439.png)

## Finding & Remediation

### Information Disclosure
#### Description
The web server exposes installation files and documentation related to CodeIgniter and Jorani through publicly accessible directories. These files disclose information about the installed applications, including default credentials for Jorani.
#### Impact
An attacker can use the disclosed information to identify installed applications and discover potential vulnerabilities. The obtained credentials lead to unauthorized access to the Jorani application that is vulnerable to a public exploit.
#### Remediation
Remove unnecessary installation files and documentation from publicly accessible directories. Disable directory listing and restrict access to sensitive application files.
### Use of Default Credentials
#### Description
The Jorani application was configured with the default credentials `bbalet:bbalet`, which were disclosed in its installation documentation.
#### Impact
An attacker can use publicly known default credentials to authenticate to the application and gain unauthorized access to its functionality.
#### Remediation
Replace all default credentials with strong, unique passwords during installation and ensure that default accounts are disabled or secured before deploying applications.
### Jorani Path Traversal (CVE-2023-26469)
#### Description
The Jorani v1.0.0 application is vulnerable to CVE-2023-26469. A publicly available exploit combines path traversal and unrestricted file upload functionality to upload a malicious file and execute it on the server.
#### Impact
An authenticated attacker can exploit these vulnerabilities to obtain a reverse shell with the privileges of the web application process.
#### Remediation
Upgrade Jorani to version 1.0.2 or later. Restrict file uploads to authorized file types, validate file paths, and prevent uploaded files from being executed by the web server.
### Unrestricted File Upload
#### Description
The Jorani application allows the upload of files that can be used to execute arbitrary code when combined with the path traversal vulnerability.
#### Impact
An attacker can upload malicious files and potentially execute arbitrary commands on the server, leading to remote code execution.
#### Remediation
Implement strict server-side file upload validation, store uploaded files outside executable web directories, and prevent the execution of uploaded content.
### Sudo Misconfiguration
#### Description
The `jordak` user is allowed to execute `/usr/bin/env` with `sudo` without providing a password. Since `env` can execute arbitrary commands, this configuration allows the user to execute commands with elevated privileges.
The user is also permitted to execute any command using `sudo` with a password, granting unrestricted administrative access when the user's password is known.
#### Impact
An attacker with access to the `jordak` account can abuse the unrestricted execution of `/usr/bin/env` to spawn a shell with `root` privileges, resulting in complete system compromise.
#### Remediation
Remove the passwordless `sudo` permission for `/usr/bin/env` and restrict privileged command execution to explicitly required commands. Review whether the user requires unrestricted `(ALL:ALL)` sudo permissions and apply the principle of least privilege.