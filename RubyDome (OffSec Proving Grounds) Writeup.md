# RubyDome (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show SSH running on port 22 and WEBrick httpd running on port 3000.
![](assets/rubydome/Pasted%20image%2020261111224920.png)

The web application converts HTML pages to PDFs.
![](assets/rubydome/Pasted%20image%2020261111224958.png)

Google research leads to CVE-2022-25765, a command injection vulnerability in PDFKit, which is a Ruby gem commonly used for converting websites or HTML to PDF documents.
![](assets/rubydome/Pasted%20image%2020261111225221.png)
## Initial Access
The exploit generates a URL containing a reverse-shell payload.
![](assets/rubydome/Pasted%20image%2020261111225301.png)

The malicious URL is sent to the web application through Burp Suite.
![](assets/rubydome/Pasted%20image%2020261111225339.png)

A reverse shell is then established as the user `andrew`.
![](assets/rubydome/Pasted%20image%2020261111225356.png)
## Privilege Escalation
The `andrew` user is permitted to run `/usr/bin/ruby /home/andrew/app/app.rb` with sudo without providing a password.
![](assets/rubydome/Pasted%20image%2020261111225629.png)

Since the `app.rb` file is located in `andrew`'s home directory and owned by the user, it can be overwritten with malicious code instead. The `echo` command is used to replace the content of the file with `exec '/bin/bash'`. The sudo command is then executed to spawn a root shell.
![](assets/rubydome/Pasted%20image%2020261111225719.png)
## Finding & Remediation
### PDFKit Command Injection (CVE-2022-25765)
#### Description
The web application uses a vulnerable version of the PDFKit Ruby gem, which is affected by CVE-2022-25765. The vulnerability allows attackers to inject operating system commands through specially crafted URLs passed to PDFKit during PDF generation.
#### Impact
An unauthenticated attacker can exploit the vulnerability to execute arbitrary operating system commands on the server. In this case, the vulnerability was exploited to establish a reverse shell as the `andrew` user.
#### Remediation
Upgrade the PDFKit Ruby gem to version 0.8.7.2 or later, which addresses CVE-2022-25765. Validate and restrict user-controlled URLs before passing them to PDFKit for processing.
### Sudo Misconfiguration
#### Description
The `andrew` user is permitted to execute `/usr/bin/ruby /home/andrew/app/app.rb` as `root` through sudo without requiring a password. The `app.rb` file is owned by `andrew` and can be modified by the same user, allowing arbitrary Ruby code to be executed with elevated privileges.
#### Impact
An attacker with access to the `andrew` account can modify the `app.rb` file to execute arbitrary commands as `root`. Since the sudo command does not require a password, the attacker can escalate privileges without knowing the account's password, resulting in complete system compromise.
#### Remediation
Remove the unnecessary sudo permission or ensure that scripts executed with elevated privileges are owned by `root` and cannot be modified by unprivileged users. Avoid granting passwordless sudo access unless strictly necessary and review sudo rules to prevent privilege escalation through user-controlled files.
