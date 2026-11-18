# Press (OffSec Proving Grounds) Writeup
## Enumeration
#### Nmap Scan Results
Nmap scan results reveal SSH running on port 22 and Apache web servers running on ports 80 and 8089. The `http-generator` field shown under the Apache web server on port 8089 reveals that the application is FlatPress fp-1.2.1.
![](assets/press/Pasted%20image%2020261111233309.png)

The application appears to use default credentials, as authenticating with `admin:password` redirects to the administration area.
![](assets/press/Pasted%20image%2020261111233849.png)

An `Uploader` area is found containing a file upload feature.
![](assets/press/Pasted%20image%2020261111234030.png)

A Google search leads to a GitHub issue describing a file upload bypass vulnerability in FlatPress 1.2.1. By adding `GIF89a;` to the beginning of a PHP file, the upload restrictions can be bypassed, allowing the file to be uploaded and stored in `Media Manager`.
![](assets/press/Pasted%20image%2020261111234155.png)
## Initial Access
The web shell is created as shown in the proof of concept.
![](assets/press/Pasted%20image%2020261111234324.png)

The file is then successfully uploaded to the web application.
![](assets/press/Pasted%20image%2020261111234357.png)

The file can be accessed through the `Media Manager` tab inside `Uploader`.
![](assets/press/Pasted%20image%2020261111234422.png)

The file is accessed by clicking the filename, which opens it in the browser, and the `whoami` command is executed. The output reveals that the application runs as `www-data`.
![](assets/press/Pasted%20image%2020261111234552.png)

To upgrade the web shell, a Bash reverse shell one-liner is encoded in Base64. Double spaces are added before the `>&` and `0>&1` operators to prevent `+` characters from appearing in the encoded output, as the web application interprets these characters as spaces.
![](assets/press/Pasted%20image%2020261111234647.png)

The following command is then sent through the web shell, utilizing `echo` to pipe the string to `base64 -d`, before piping it again to `bash`.
![](assets/press/Pasted%20image%2020261111234824.png)

This results in a reverse shell as `www-data`.
![](assets/press/Pasted%20image%2020261111234854.png)
## Privilege Escalation
The `www-data` user is permitted to execute `/usr/bin/apt-get` with `sudo` without providing a password.
![](assets/press/Pasted%20image%2020261111234926.png)


GTFOBins documents a privilege escalation technique using `apt-get changelog apt`, which opens the changelog in the `less` pager. When executed with elevated privileges, the pager also runs with those privileges.
![](assets/press/Pasted%20image%2020261111234952.png)

The command is executed, displaying the changelog in the `less` pager as expected. The `/bin/bash` command is then executed directly from the pager, resulting in a root shell.
![](assets/press/Pasted%20image%2020261111235135.png)
## Finding & Remediation
### Use of Default Credentials
#### Description
The FlatPress application is configured with default administrative credentials (`admin:password`), allowing unauthorized users to authenticate to the administration interface using publicly known credentials.
#### Impact
An attacker can use the default credentials to gain administrative access to the FlatPress application, including its file upload functionality.
#### Remediation
Replace the default administrative credentials with a strong, unique password. Disable or remove unused default accounts and restrict access to the administration interface to authorized users.
### Unrestricted File Upload
#### Description
FlatPress 1.2.1 contains a file upload validation bypass that allows PHP files to be uploaded by prepending the `GIF89a;` signature. The application accepts these files through the `Uploader` functionality and stores them in a location where they can be accessed and executed through the web server.
#### Impact
An attacker with access to the file upload functionality can upload a malicious PHP file and execute arbitrary commands on the server. In this case, the vulnerability was exploited to establish a reverse shell as the `www-data` user.
#### Remediation
Upgrade FlatPress to a version that addresses the file upload validation bypass. Implement server-side file validation using an allowlist of permitted extensions and verify file contents rather than relying solely on file signatures. Store uploaded files outside the web root or configure the upload directory to prevent script execution.
### Sudo Misconfiguration
#### Description
The `www-data` user is permitted to execute `/usr/bin/apt-get` as `root` through sudo without requiring a password. The `apt-get changelog` functionality can launch an interactive `less` pager, which allows shell commands to be executed with the privileges of the invoking process.
#### Impact
An attacker with access to the `www-data` account can exploit the unrestricted pager to execute arbitrary commands as `root`. Since the sudo command does not require a password, privilege escalation is possible without knowing the account's password, resulting in complete system compromise.
#### Remediation
Remove the unnecessary sudo permission for `/usr/bin/apt-get` from the `www-data` account. If elevated package management operations are required, restrict them to trusted administrative users and avoid allowing unprivileged service accounts to execute commands that can launch interactive pagers or arbitrary subprocesses with root privileges.
