# Sea (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show FTP running on port 21 with anonymous login enabled, SSH running on port 22 and Apache web servers running on ports 80 and 55743. Nmap is also able to list files inside of FTP, revealing four log files.
![](assets/sea/Pasted%20image%2020261111215011.png)

The log files reveal the path to a database file located at `/var/www/SeaCMS/th4o4p/`, as well as a POST request to `/th4o4p/database.php` using the `cfg_dbpwd` parameter in the URL that results in an error.
![](assets/sea/Pasted%20image%2020261111215801.png)

The web application on port 55743 displays a page entirely in Chinese.
![](assets/sea/Pasted%20image%2020261111215433.png)

When translating the file, the bottom of the page displays `Ocean CMS`. This could be a translation error since the path to the database file mentioned SeaCMS.
![](assets/sea/Pasted%20image%2020261111215716.png)

## Initial Access
An attempt to send the request found in the log file as a GET request results in the server's response containing user credentials.
![](assets/sea/Pasted%20image%2020261111220229.png)

These credentials are used for SSH authentication as `nicholas`.
![](assets/sea/Pasted%20image%2020261111220305.png)
## Privilege Escalation
Enumeration of the SeaCMS directory on the target reveals a second `database.php` file containing a different password for the user `nicholas`.
![](assets/sea/Pasted%20image%2020261111220432.png)

The `su` command is used to test the password against the `root` account, successfully switching to the account and confirming that this is the `root` user's password.
![](assets/sea/Pasted%20image%2020261111220506.png)
## Finding & Remediation
### Anonymous FTP Access
#### Description
The FTP service allows anonymous authentication and provides access to log files containing sensitive information, including internal file paths and HTTP requests.
#### Impact
An attacker can access the log files without valid credentials and obtain information about the application's internal structure and endpoints. In this case, the exposed logs revealed a request to a `database.php` file that led to the disclosure of user credentials.
#### Remediation
Disable anonymous FTP access unless explicitly required. Restrict access to log files to authorized users and ensure that sensitive application information is not exposed through publicly accessible services.
### Sensitive Credential Disclosure
#### Description
The SeaCMS application exposes credentials through a `database.php` endpoint that returns sensitive information in response to a GET request. An additional `database.php` file contains credentials in plaintext, including a password that is also used by the operating system's `root` account.
#### Impact
An attacker can retrieve exposed credentials and use them to gain unauthorized SSH access to the system. The reuse of an exposed password for the `root` account also allows an attacker to escalate privileges and gain complete control over the system.
#### Remediation
Prevent unauthorized access to configuration files and ensure that sensitive files cannot be retrieved through HTTP requests. Store credentials securely with restrictive file permissions, remove unnecessary copies of configuration files and avoid reusing passwords across application and operating system accounts. Rotate all exposed credentials, including the `root` password.
