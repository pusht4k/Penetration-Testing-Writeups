# Vmdak (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show FTP running on port 21 with anonymous login enabled and a `config.xml` file listed. Additionally, SSH is running on port 22 and Apache web servers run on ports 80 and 9443.
![](assets/vmdak/Pasted%20image%2020261115220029.png)

The `config.xml` file is downloaded from FTP and inspected, revealing an installation of Jenkins version 2.401.2 and the path to the `initialAdminPassword`.
![](assets/vmdak/Pasted%20image%2020261115220437.png)

The web application running on port 9443 contains an `Admin Dashboard` button that leads to a login page.
![](assets/vmdak/Pasted%20image%2020261115220544.png)

The dashboard is accessed through an SQL injection authentication bypass method using the `admin'-- -` username without providing a password.
![](assets/vmdak/Pasted%20image%2020261115220636.png)

The dashboard features a file upload feature intended for user profile pictures. The images are stored in the `/uploadImage/Profile/` directory and can be directly accessed.
![](assets/vmdak/Pasted%20image%2020261115221511.png)
## Initial Access
File validation prevents direct uploads of PHP files. To bypass this, a file can be initially created with placeholder characters (such as `AAAA`) at the beginning. These characters are then modified using a hex editor to match the JPEG magic bytes (`FF D8 FF E0`).
The following web shell PHP script is created, accepting the `cmd` parameter in the URL of a GET request and executing its value on the underlying system with the `system()` function.
![](assets/vmdak/Pasted%20image%2020261115221602.png)

The file is saved as `simple.jpg` and `hexeditor` is used to change the first letters to `FF D8 FF E0` which corresponds to JPEG.
![](assets/vmdak/Pasted%20image%2020261115221754.png)

The upload request is then intercepted and the filename is changed to `simple.php`.
![](assets/vmdak/Pasted%20image%2020261115221902.png)

The web application does not return an error, and the profile image for the compromised `admin` account disappears, displaying a `user image` text.
![](assets/vmdak/Pasted%20image%2020261115222028.png)

Right-clicking the `user image` text displays a menu containing the `Open Image in New Tab` option.
![](assets/vmdak/Pasted%20image%2020261115222120.png)

Selecting that option redirects to the `simple.php` file. The `whoami` command is executed, revealing the application is running as `www-data`.
![](assets/vmdak/Pasted%20image%2020261115222255.png)

In order to upgrade the web shell, a Bash reverse shell one-liner is encoded in Base64.
![](assets/vmdak/Pasted%20image%2020261115222351.png)

The following command is then sent in the `cmd` parameter. It uses `echo` to pipe the encoded string to `base64 -d`, then pipes the decoded string to `bash` for execution.
![](assets/vmdak/Pasted%20image%2020261115222423.png)

This results in a reverse shell as `www-data`.
![](assets/vmdak/Pasted%20image%2020261115222525.png)
## Privilege Escalation
The `pspy` tool reveals that the Jenkins application is running on port 8080 as UID=0 (`root`).
![](assets/vmdak/Pasted%20image%2020261115222551.png)

A `chisel` Linux binary is transferred to the target and executed to establish a port forwarding tunnel, mapping port 1337 on the attacking machine to port 8080 on the target.
![](assets/vmdak/Pasted%20image%2020261115222800.png)

A `chisel` server on the attack host catches the connection and establishes the tunnel.
![](assets/vmdak/Pasted%20image%2020261115222731.png)

Direct access to the tunneled port reveals that Jenkins has not been configured yet, requiring a password saved in the `initialAdminPassword` file in `/root/.jenkins/secrets/`.
![](assets/vmdak/Pasted%20image%2020261115222821.png)

The previous `config.xml` file listed Jenkins' version as 2.401.2. This version is vulnerable to arbitrary file read (CVE-2024-23897) with an available proof of concept on [anantshri's GitHub](https://github.com/godylockz/CVE-2024-23897/blob/main/jenkins_fileread.py).
**![](assets/vmdak/Pasted%20image%2020261115223217.png)

The script is used to read the `initialAdminPassword` file.
![](assets/vmdak/Pasted%20image%2020261115223308.png)

The password is accepted and the application proceeds with the configuration process.
![](assets/vmdak/Pasted%20image%2020261115223337.png)

Jenkins is known for having a Script Console endpoint, which allows the execution of Groovy scripts to automate tasks such as plugin installation, configuration parameter changes, and more. This also enables the execution of system commands through Groovy scripts.
The Script Console is found at the `/script` endpoint on the web server. A reverse shell script written in Groovy is executed from the console.
![](assets/vmdak/Pasted%20image%2020261115223448.png)

This establishes a reverse shell running as the `root` user.
![](assets/vmdak/Pasted%20image%2020261115223531.png)

## Finding & Remediation
### Anonymous FTP Access
#### Description
The FTP service allows anonymous authentication and provides access to a `config.xml` file containing internal Jenkins configuration information, including the installed version and the location of the initial administrative password.
#### Impact
An unauthenticated attacker can obtain sensitive configuration information and identify potentially vulnerable software versions.
#### Remediation
Disable anonymous FTP access unless explicitly required. Remove sensitive configuration files from publicly accessible directories and restrict access to authorized users.
### SQL Injection Authentication Bypass
#### Description
The administrative login page is vulnerable to SQL injection due to improper handling of user-controlled input in authentication queries.
#### Impact
An unauthenticated attacker can bypass the authentication mechanism and gain unauthorized access to the administrative dashboard.
#### Remediation
Use parameterized SQL queries for all authentication operations. Validate user-controlled input and ensure that authentication decisions cannot be manipulated through SQL syntax.
### Unrestricted File Upload
#### Description
The profile picture upload functionality relies on insufficient file validation. Uploaded files are stored in a publicly accessible directory where PHP scripts can be executed.
#### Impact
An attacker with access to the administrative dashboard can upload malicious files and achieve arbitrary code execution. In this case, exploitation resulted in a reverse shell as `www-data`.
#### Remediation
Enforce strict file extension and content validation. Generate server-side filenames, re-encode uploaded images and store them outside the web root. Disable script execution in upload directories.
### Jenkins Arbitrary File Read (CVE-2024-23897)
#### Description
Jenkins version 2.401.2 is vulnerable to an arbitrary file read vulnerability in its CLI argument parsing mechanism.
#### Impact
An attacker can access sensitive files readable by the Jenkins process. In this case, exploitation exposed the initial administrative password, resulting in unauthorized administrative access to Jenkins.
#### Remediation
Upgrade Jenkins to a supported release containing the fix introduced in version 2.442, LTS 2.426.3 or LTS 2.440.1. If an immediate upgrade is not possible, disable CLI access as a temporary mitigation. Rotate exposed credentials and restrict access to Jenkins to authorized users.
### Excessive Service Privileges
#### Description
The Jenkins application runs with root privileges instead of using a dedicated, unprivileged service account.
#### Impact
Compromise of the Jenkins application can result in full system compromise due to the privileges inherited by processes executed through Jenkins. In this case, administrative access to Jenkins resulted in a root shell.
#### Remediation
Run Jenkins under a dedicated, unprivileged service account. Apply the principle of least privilege and restrict access to sensitive system files and administrative functionality.
