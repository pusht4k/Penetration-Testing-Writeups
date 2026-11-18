# Sau (HackTheBox) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal that SSH is running on port 22 and a Golang HTTP Server is running on port 55555 with the http-title "Request Baskets". Ports 80 and 8338 are shown as filtered.
![](assets/sau/Pasted%20image%2020261109213237.png)

A Google search reveals that port 8338 is commonly used as the default server port for Mailtrail.
![](assets/sau/Pasted%20image%2020261109213339.png)


A subsequent Google search about the Request Baskets application running on port 55555, identified as version 1.2.1, reveals that it is vulnerable to SSRF (CVE-2023-27163).
![](assets/sau/Pasted%20image%2020261109213608.png)
## Initial Access
The basket configuration requires a URL to forward requests to, while the `Proxy Response` option can be enabled to return responses, effectively turning the basket into a proxy. The `Forward URL` is set to `http://127.0.0.1:8443` to interact with the Mailtrail service, with `Proxy Response` enabled.
![](assets/sau/Pasted%20image%2020261109213720.png)

Mailtrail is identified as running version 0.53, which is vulnerable to an unauthenticated remote code execution vulnerability (CVE-2025-34073). A publicly available proof of concept by Iyaad Luqman is found and downloaded to the attacking machine.
![](assets/sau/Pasted%20image%2020261109214326.png)

The exploit is executed against the basket's IP address, pointing to port 443 on the attacking machine.
![](assets/sau/Pasted%20image%2020261109214442.png)

This results in a reverse shell running as the user `puma`.
![](assets/sau/Pasted%20image%2020261109214516.png)
## Privilege Escalation
The `sudo -l` command reveals that the compromised user can run `/usr/bin/systemctl status trail.service` using `sudo` without providing a password.
![](assets/sau/Pasted%20image%2020261109214611.png)

Executing the permitted `sudo` command opens the `trail.service` status output in the `less` pager. Since `less` is running with root privileges, entering `!/bin/sh` within the pager spawns a shell as `root`.
![](assets/sau/Pasted%20image%2020261109214742.png)


## Findings & Remediation

### Server Side Request Forgery (CVE-2023-27163)
#### Description
The Request Baskets application is running version 1.2.1, which is vulnerable to Server-Side Request Forgery (CVE-2023-27163). The vulnerability allows users to configure baskets to forward HTTP requests to arbitrary URLs, including internal services that are not directly accessible from external networks.
#### Impact
An attacker can exploit the vulnerability to access internal services and potentially retrieve sensitive information. In this case, the vulnerability was used to forward requests to the internal Maltrail service, allowing the attacker to interact with an otherwise inaccessible application.
#### Remediation
Since no officially confirmed patched version is available for CVE-2023-27163, consider replacing Request Baskets with a maintained alternative. If replacement is not possible, restrict access to the application, implement strict validation of forwarding destinations, and prevent requests from reaching internal or restricted network addresses.
### Mailtrail 0.53 Remote Code Execution (CVE-2025-34073)
#### Description
The Maltrail application is running version 0.53, which is vulnerable to unauthenticated command injection (CVE-2025-34073). The vulnerability exists in the `/login` endpoint, where the `username` parameter is passed to the `subprocess.check_output()` function without sufficient sanitization, allowing arbitrary operating system commands to be injected.
#### Impact
An unauthenticated attacker who can reach the vulnerable service can execute arbitrary operating system commands with the privileges of the Maltrail process. In this case, the vulnerability was exploited through the Request Baskets SSRF vulnerability to obtain a reverse shell as the `puma` user.
#### Remediation
Upgrade Maltrail to version 0.55 or later, which addresses the command injection vulnerability. Additionally, restrict communication between internal services to prevent unauthorized access through vulnerabilities such as SSRF, and ensure that Mailtrail runs under a dedicated, minimally privileged account.
### Sudo Misconfiguration
#### Description
The `puma` user is permitted to execute `/usr/bin/systemctl status trail.service` using `sudo` without providing a password. The command displays its output through the `less` pager, which allows users to execute shell commands.
#### Impact
A local attacker with access to the `puma` account can execute the permitted command and use the `less` pager to spawn a shell with root privileges, resulting in full system compromise.
#### Remediation
If the `puma` user does not require this sudo permission, it should be removed. Otherwise, the permission should require a password. Additionally, secure pager mode should be enforced using `SYSTEMD_PAGERSECURE=1`, or the pager should be disabled entirely using `--no-pager`, to prevent shell execution through `less` even if the user successfully authenticates with sudo. Ensure that these restrictions cannot be overridden by the user.

