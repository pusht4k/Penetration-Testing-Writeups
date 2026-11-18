# Ochima (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show SSH running on port 22, an Apache web server running on port 80, and Mailtrail version 0.52 running on port 8338.
![](assets/ochima/Pasted%20image%2020261111181618.png)

A Google search leads to a web page on Rapid7 describing an unauthenticated command injection vulnerability (CVE-2025-34073).
![](assets/ochima/Pasted%20image%2020261111181716.png)

The description reveals that the `username` parameter is vulnerable to command injection.
![](assets/ochima/Pasted%20image%2020261111181931.png)

Additional research shows that commands can be injected by using a semicolon in the username parameter before wrapping the command in backticks.
![](assets/ochima/Pasted%20image%2020261111182226.png)
## Initial Access
The home page of the web application on port 8338 is the Mailtrail login page.
![](assets/ochima/Pasted%20image%2020261111182011.png)

A login request is intercepted using Burp Suite. The request is then sent to Burp Repeater.
![](assets/ochima/Pasted%20image%2020261111182051.png)

A Bash reverse shell one-liner is encoded in Base64.
![](assets/ochima/Pasted%20image%2020261111182333.png)

The `username` field is modified to execute `echo`, piping the encoded string to `base64 -d` for decoding and then to `bash` for execution.
![](assets/ochima/Pasted%20image%2020261111182354.png)

The request is then sent to the web application, executing the code and establishing a reverse shell as the user `snort`.
![](assets/ochima/Pasted%20image%2020261111182447.png)
## Privilege Escalation
The `pspy32` tool finds a cron job executing `/var/backups/etc_Backup.sh` as `UID=0` (`root`).
![](assets/ochima/Pasted%20image%2020261111182647.png)

The backup script is writable by all users on the `ochima` host.
![](assets/ochima/Pasted%20image%2020261111182728.png)

The file is overwritten with a malicious shell command, which copies `/bin/bash` to `/home/snort/rootbash` and sets the SUID bit on the copied binary.
![](assets/ochima/Pasted%20image%2020261111182811.png)

Shortly after, the cron job executes the malicious script and the `rootbash` binary is found in `snort`'s home directory.
![](assets/ochima/Pasted%20image%2020261111182858.png)

An elevated shell is obtained by executing `rootbash` with the `-p` parameter to maintain root privileges.
![](assets/ochima/Pasted%20image%2020261111182942.png)
## Finding & Remediation
### Maltrail Command Injection (CVE-2025-34073)
#### Description
The target runs Maltrail version 0.52, which is vulnerable to CVE-2025-34073. The vulnerability allows unauthenticated attackers to inject arbitrary operating system commands through the `username` parameter in POST requests to the `/login` endpoint due to improper handling of user-supplied input.
#### Impact
An unauthenticated attacker can exploit this vulnerability to execute arbitrary commands on the target system and establish a reverse shell as the `snort` user.
#### Remediation
Apply a verified security fix for CVE-2025-34073 that prevents user-controlled input from being passed to shell commands without proper validation. Avoid shell-based command execution where possible and restrict access to the Maltrail web interface to trusted networks.
### Misconfigured Cron Job
#### Description
A cron job executes `/var/backups/etc_Backup.sh` with `root` privileges. The script is writable by all users, allowing lower-privileged accounts to modify the commands executed by the cron job.
#### Impact
An attacker with access to the `snort` account can replace the script's contents with malicious commands that execute as `root`. In this case, the script was modified to create a SUID-enabled copy of `/bin/bash`, allowing the attacker to obtain a root shell.
#### Remediation
Restrict ownership and write permissions on `/var/backups/etc_Backup.sh` to trusted administrative accounts. Ensure that cron jobs running with elevated privileges do not execute scripts that can be modified by lower-privileged users.
