# Twiggy (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show SSH running on port 22, Nginx running on port 80, ZeroMQ ZMTP 2.0 running on ports 4505 and 4506 and a second Nginx web application running on port 8000.
![](assets/twiggy/Pasted%20image%2020261115225415.png)

The web application on port 80 features a Mezzanine application and a login page. Attempted logins with easy-to-guess credentials fail.
![](assets/twiggy/Pasted%20image%2020261115225506.png)

The web application on port 8000 returns data in JSON format. 
![](assets/twiggy/Pasted%20image%2020261115225544.png)

A Google search for `runner_async` suggests that SaltStack might be used.
![](assets/twiggy/Pasted%20image%2020261115225636.png)

Further research leads to a proof-of-concept exploit for remote code execution in SaltStack 3000.1 (CVE-2020-11651) on Exploit-DB.
![](assets/twiggy/Pasted%20image%2020261115225806.png)
## Initial Access
The exploit's `-p` parameter is set by default to port 4506, which is open on the target machine.
![](assets/twiggy/Pasted%20image%2020261115225835.png)

The exploit script is used to execute a Bash reverse shell one-liner that connects back to port 80 on the attack host.
![](assets/twiggy/Pasted%20image%2020261115225906.png)

This attempt is successful and a reverse shell is established running as the `root` user.
![](assets/twiggy/Pasted%20image%2020261115230051.png)
## Privilege Escalation
Privilege escalation is not required for this lab as the initial exploit compromises the `root` account directly.
## Finding & Remediation
### SaltStack Unauthenticated Remote Code Execution (CVE-2020-11651)
#### Description
SaltStack 3000.1 is vulnerable to an authentication bypass due to insufficient validation of method calls within the salt-master process. The vulnerability exposes administrative functionality to unauthenticated remote users.
#### Impact
An unauthenticated attacker can execute arbitrary commands with elevated privileges. In this case, exploitation of the vulnerability resulted in a reverse shell as the `root` user, granting full control over the target.
#### Remediation
Upgrade SaltStack to a supported release containing the security fixes introduced in versions 3000.2 and 2019.2.4. Restrict network access to the Salt master service on ports 4505 and 4506 to trusted systems and regularly apply security updates.