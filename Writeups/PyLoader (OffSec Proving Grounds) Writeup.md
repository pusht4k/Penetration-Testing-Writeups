# PyLoader (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan finds two open TCP ports on the target, with SSH running on port 22 and a CherryPy wsgiserver running on port 9666. Nmap also shows the `http-server-header` being Cheroot/8.6.0 with the `http-title` being pyLoad.
![](assets/pyloader/Pasted%20image%2020261111230650.png)

Searchsploit shows a pre-authentication remote code execution exploit for pyLoad (CVE-2023-0297).
![](assets/pyloader/Pasted%20image%2020261111230810.png)
## Initial Access
While the version of the service was not identified through enumeration of the web application, the exploit is tested with the `whoami` command. The output confirms that the command has been executed on the target machine.
![](assets/pyloader/Pasted%20image%2020261111230956.png)

Next, a Bash reverse shell one-liner is encoded in Base64. The encoded string is then sent in a command that utilizes `echo` to pipe it to `base64 -d`, before piping it again to `bash` for execution. This causes the terminal to hang, which is the expected behavior.
![](assets/pyloader/Pasted%20image%2020261111231131.png)

A reverse shell is established as `root`, with `proof.txt` retrieved from the user's home directory.
![](assets/pyloader/Pasted%20image%2020261111231254.png)
## Privilege Escalation
Privilege escalation is not required for this lab as the initial exploit compromises the `root` account.

## Finding & Remediation
### pyLoad Remote Code Execution (CVE-2023-0297)
#### Description
The pyLoad application is vulnerable to CVE-2023-0297, a pre-authentication remote code execution vulnerability caused by improper input validation in the `js2py` functionality. The vulnerability allows unauthenticated attackers to execute arbitrary operating system commands through specially crafted HTTP requests.
#### Impact
An unauthenticated attacker can exploit the vulnerability to execute arbitrary commands with the privileges of the pyLoad process. In this case, the application was running as `root`, allowing the attacker to establish a reverse shell with root privileges and gain complete control over the system.
#### Remediation
Upgrade pyLoad to version 0.5.0b3.dev31 or later, which addresses CVE-2023-0297. Run the pyLoad service under a dedicated, unprivileged account instead of `root` to limit the impact of potential vulnerabilities.
