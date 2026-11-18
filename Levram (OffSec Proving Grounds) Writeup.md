# Levram (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal that SSH is running on port 22 and a WSGIServer version 0.2 is running on port 8000. Nmap also displays the `http-title` Gerapy, indicating that the Gerapy web application is installed.
![](assets/levram/Pasted%20image%2020261111004700.png)

The Gerapy web application on port 8000 is configured with the default credentials `admin:admin`. Using these credentials results in a redirection to the dashboard.
![](assets/levram/Pasted%20image%2020261111004642.png)
## Initial Access
The application is identified as version 0.9.7, which is vulnerable to remote code execution (CVE-2021-43857). A proof of concept script is available on Exploit-DB with [EDB-ID 50640](https://www.exploit-db.com/exploits/50640).
![](assets/levram/Pasted%20image%2020261111004755.png)

The exploit is successful, establishing a reverse shell as the `app` user.
![](assets/levram/Pasted%20image%2020261111005017.png)
## Privilege Escalation
Enumeration of capabilities reveals that `/usr/bin/python3.10` is configured with the `cap_setuid=ep` capability.
![](assets/levram/Pasted%20image%2020261111005118.png)

GTFOBins documents a privilege escalation vector for this capability. The provided Python command uses the `os` library and the `setuid()` function to set the process UID to 0 (`root`) before executing `/bin/sh`.
![](assets/levram/Pasted%20image%2020261111005138.png)

Using the exact command results in a `root` shell.
![](assets/levram/Pasted%20image%2020261111005342.png)
## Finding & Remediation
### Use of Default Credentials
#### Description
The Gerapy web application is configured with the default administrative credentials `admin:admin`, allowing access to the dashboard without requiring a unique password.
#### Impact
An unauthenticated attacker can use the default credentials to gain administrative access to the Gerapy application.
#### Remediation
Replace the default administrative password with a strong, unique password. Ensure that default credentials are changed during the initial deployment of the application.
### Gerapy Remote Code Execution (CVE-2021-43857)
#### Description
The target runs Gerapy version 0.9.7, which is vulnerable to CVE-2021-43857. The vulnerability allows authenticated attackers to execute arbitrary operating system commands through the application's project management functionality.
#### Impact
An attacker with valid administrative credentials can exploit this vulnerability to execute arbitrary commands and establish a reverse shell as the `app` user.
#### Remediation
Upgrade Gerapy to version 0.9.8 or later, which addresses CVE-2021-43857. Restrict access to the Gerapy administrative interface to trusted users.
### Misconfigured Linux Capability
#### Description
The `/usr/bin/python3.10` binary is configured with the `cap_setuid=ep` capability, allowing Python processes to change their user ID, including to UID 0 (`root`).
#### Impact
An attacker with access to the `app` account can exploit this capability to execute a shell with `root` privileges, resulting in complete system compromise.
#### Remediation
Remove the unnecessary capability from the Python binary using `sudo setcap -r /usr/bin/python3.10`. Grant elevated capabilities only to binaries that explicitly require them and regularly audit file capabilities to prevent privilege escalation.

