# law (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan shows SSH running on port 22 and an Apache web server running on port 80.
![](assets/law/Pasted%20image%2020261110172546.png)

The web application in use is htmLawed version 1.2.5.
![](assets/law/Pasted%20image%2020261110172610.png)

Searchsploit displays a remote code execution vulnerability (CVE-2022-35914) affecting this version of htmLawed.
![](assets/law/Pasted%20image%2020261110172732.png)
## Initial Access
The exploit is tested, executing `curl` to connect to port 443 on the attacking machine.
![](assets/law/Pasted%20image%2020261110172855.png)

Netcat receives a connection, confirming that the application is vulnerable.
![](assets/law/Pasted%20image%2020261110172916.png)

To execute a Bash reverse shell one-liner, the command is first encoded in Base64.
![](assets/law/Pasted%20image%2020261110172955.png)

The following command is executed on the web application, using `echo` to pass the encoded string to `base64 -d` through a pipe, before passing it through a second pipe to `bash`.
![](assets/law/Pasted%20image%2020261110173111.png)

A reverse shell is then established as the `www-data` user.
![](assets/law/Pasted%20image%2020261110173204.png)
## Privilege Escalation
A `cleanup.sh` file that is owned by `www-data` is found in `/var/www/`.
![](assets/law/Pasted%20image%2020261110173252.png)

The script deletes the `access.log` and `error.log` files from the `apache2` directory.
![](assets/law/Pasted%20image%2020261110173333.png)

While initial enumeration of cron jobs does not reveal an automated task running this script, the `pspy32` tool reveals that the script is executed every minute with `UID=0`, which translates to `root`.
![](assets/law/Pasted%20image%2020261110173441.png)

The cleanup script is overwritten with a Bash reverse shell one-liner pointing to port 4444 on the attacking machine.
![](assets/law/Pasted%20image%2020261110173627.png)

A minute later, the script is automatically executed, leading to a reverse shell with root privileges.
![](assets/law/Pasted%20image%2020261110173756.png)
## Finding & Remediation
### CVE-2022-35914 - htmLawed Remote Code Execution
#### Description
The web application uses htmLawed version 1.2.5, which is vulnerable to CVE-2022-35914. The vulnerability allows unauthenticated attackers to execute arbitrary PHP code through the exposed `htmLawedTest.php` functionality.
#### Impact
An unauthenticated attacker can exploit this vulnerability to execute arbitrary commands on the target system and establish a reverse shell as the `www-data` user.
#### Remediation
Upgrade htmLawed to version 1.2.9 or later. Remove the `htmLawedTest.php` file from production environments and restrict public access to unnecessary testing functionality.
### Misconfigured Cron Job
#### Description
A cron job executes `/var/www/cleanup.sh` every minute with `root` privileges. Both the script and its parent directory are controlled by the `www-data` user, allowing the script to be modified or replaced by a lower-privileged account.
#### Impact
An attacker with access to the `www-data` account can replace the cleanup script with malicious commands that are automatically executed with `root` privileges, resulting in complete system compromise.
#### Remediation
Restrict ownership and write permissions on `/var/www/cleanup.sh` and its parent directory to trusted administrative accounts. Ensure that cron jobs executed with elevated privileges do not rely on files or directories that can be modified by lower-privileged users.
