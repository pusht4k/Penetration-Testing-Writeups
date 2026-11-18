# Plum (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal SSH running on port 22 and an Apache web server running on port 80.
![](assets/plum/Pasted%20image%2020261111163219.png)

The web application is PluXml and an administration page button is found at the bottom of the home page.
![](assets/plum/Pasted%20image%2020261111163313.png)

The button redirects to a login page.
![](assets/plum/Pasted%20image%2020261111163342.png)

The application is configured with default credentials, as using `admin` for both the username and password successfully redirects to the PluXml dashboard. The application's version is 5.8.7, as shown in the dashboard menu on the left side of the page.
![](assets/plum/Pasted%20image%2020261111163449.png)

A Google search leads to a GitHub Advisory Database entry revealing that PluXml version 5.8.7 is vulnerable to remote code execution via crafted PHP code inserted into static pages (CVE-2022-25018).
![](assets/plum/Pasted%20image%2020261111163624.png)
## Initial Access
Static pages can be created through the `Static pages` option on the left-side menu.
![](assets/plum/Pasted%20image%2020261111163823.png)

A `Webshell` page is created, containing a simple PHP web shell which accepts the `cmd` parameter in GET requests and executes its value on the underlying system through the `system()` function.
![](assets/plum/Pasted%20image%2020261111163856.png)

The `Webshell` page is then accessed directly. The `id` command is successfully executed, revealing that the application runs as the `www-data` user.
![](assets/plum/Pasted%20image%2020261111164030.png)

To upgrade from a web shell to a reverse shell, the content of the `Webshell` page is changed to the contents of the PHP reverse shell script from PentestMonkey. The PHP script will connect back to port 443 on the attacking machine.
![](assets/plum/Pasted%20image%2020261111164144.png)

The page is then refreshed, executing the malicious code and establishing a reverse shell.
![](assets/plum/Pasted%20image%2020261111164302.png)

The user flag is found in `/var/www/`.
![](assets/plum/Pasted%20image%2020261111164752.png)
## Privilege Escalation
Enumeration of open TCP ports on the target using the `ss -tulpn` command reveals that SMTP is running on port 25.
![](assets/plum/Pasted%20image%2020261111164400.png)

Inspection of the `/var/mail` directory reveals a stored email for `www-data`.
![](assets/plum/Pasted%20image%2020261111164504.png)

The email contains plaintext credentials for the `root` user.
![](assets/plum/Pasted%20image%2020261111164601.png)

These credentials are used with the `su` command to switch to the `root` account and retrieve `/root/proof.txt`.
![](assets/plum/Pasted%20image%2020261111164643.png)

## Finding & Remediation
### Use of Default Credentials
#### Description
The PluXml application is configured with the default administrative credentials `admin:admin`, allowing access to the administration dashboard without requiring a unique password.
#### Impact
An unauthenticated attacker can use the default credentials to gain administrative access to the PluXml application.
#### Remediation
Replace the default administrative password with a strong, unique password. Ensure that default credentials are changed during the initial deployment of the application.
### PluXml 5.8.7 PHP Code Injection - CVE-2022-25018
#### Description
The target runs PluXml version 5.8.7, which is vulnerable to CVE-2022-25018. The vulnerability allows authenticated users to inject and execute arbitrary PHP code through the application's static page functionality.
#### Impact
An attacker with access to the PluXml administration dashboard can exploit this vulnerability to execute arbitrary system commands and establish a reverse shell as the `www-data` user.
#### Remediation
Disable PHP execution within user-editable static pages and restrict access to the static page editing functionality. PluXml version 5.8.8 introduced a fix for this vulnerability, but the fix was reverted in version 5.8.9. Ensure that the deployed version includes an effective fix before upgrading.
### Insecure Credential Storage
#### Description
An email stored in the `/var/mail` directory contains the `root` account password in plaintext and is accessible to the `www-data` user.
#### Impact
An attacker with access to the `www-data` account can retrieve the exposed password and authenticate as `root` using the `su` command, resulting in complete system compromise.
#### Remediation
Avoid storing privileged account credentials in plaintext emails or other files accessible to lower-privileged users. Remove the exposed credentials, rotate the `root` password, and restrict access to sensitive mailboxes and files.