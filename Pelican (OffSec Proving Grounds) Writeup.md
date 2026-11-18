# Pelican (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results suggest that the target is a Linux machine. The results also show OpenSSH running on ports 22 and 2222, Samba smbd running on ports 139 and 445, and CUPS running on port 631. Additional services are found on ports 2181 (ZooKeeper), 8080 (Jetty web server), 8081 (Nginx web server) and 46295 (Java RMI).
![](assets/pelican/Pasted%20image%2020261111170047.png)

The web application on port 8081 redirects to the Exhibitor for ZooKeeper Web UI. The top right of the page shows that it is version 1.0.
![](assets/pelican/Pasted%20image%2020261111170404.png)

Google research leads to CVE-2019-5029, which is a remote code execution vulnerability in Exhibitor Web UI versions 1.0.9 to 1.7.1. Since the web application is shown as version 1.0, it might be vulnerable.
![](assets/pelican/Pasted%20image%2020261111170609.png)

A proof of concept can be found with Searchsploit.
![](assets/pelican/Pasted%20image%2020261111170656.png)

The proof of concept details the steps to exploit this issue to establish a reverse shell.
![](assets/pelican/Pasted%20image%2020261111170738.png)
## Initial Access
The reverse shell one-liner is added to the `java.env script` field under the `Config` tab.
![](assets/pelican/Pasted%20image%2020261111170848.png)

After clicking the `Commit > Commit All At Once` option, the application displays a loading window with the `Config change in process` message.
![](assets/pelican/Pasted%20image%2020261111170928.png)

Shortly after, Netcat receives a connection.
![](assets/pelican/Pasted%20image%2020261111171035.png)

The reverse shell is running as the user `charles`.
![](assets/pelican/Pasted%20image%2020261111171111.png)

A TTY shell is spawned using Python's `pty` library. The user flag is then retrieved from `charles`' home directory.
![](assets/pelican/Pasted%20image%2020261111171845.png)

## Privilege Escalation
The `charles` user is allowed to execute `/usr/bin/gcore` using `sudo` without providing a password.
![](assets/pelican/Pasted%20image%2020261111171203.png)

This binary can be used to save a core dump file for a running process. The `ps aux` command reveals a `/usr/bin/password-store` process executed by `root`.
![](assets/pelican/Pasted%20image%2020261111171344.png)

The `gcore` binary is then used with `sudo` to generate a core dump file for the running process.
![](assets/pelican/Pasted%20image%2020261111171631.png)

Using `strings` on the dump file reveals a suggested password for the `root` user.
![](assets/pelican/Pasted%20image%2020261111171708.png)

The password is used to switch to the `root` account using `su`. The root flag is retrieved from the `root` account's home directory.
![](assets/pelican/Pasted%20image%2020261111171741.png)

## Finding & Remediation
### Exhibitor OS Command Injection (CVE-2019-5029)
#### Description
The target exposes a vulnerable Exhibitor Web UI that does not require authentication. CVE-2019-5029 allows unauthenticated attackers to inject arbitrary shell commands through the `java.env script` field in the configuration editor, which are executed when ZooKeeper is launched.
#### Impact
An unauthenticated attacker can exploit this vulnerability to execute arbitrary commands on the target and establish a reverse shell as the `charles` user.
#### Remediation
Restrict access to the Exhibitor Web UI to trusted networks using firewall rules. Implement authentication through a reverse proxy and prevent direct access to the interface. If the web interface is unnecessary, disable it and use an alternative ZooKeeper supervision mechanism.
### Sudo Misconfiguration
#### Description
The `charles` user is permitted to execute `/usr/bin/gcore` with `sudo` without providing a password. This allows the user to generate core dumps of processes running with elevated privileges, including processes owned by `root`.
#### Impact
An attacker with access to the `charles` account can use `gcore` to extract sensitive information from privileged processes, including credentials stored in their memory.
#### Remediation
Remove `/usr/bin/gcore` from the `charles` user's sudo permissions. Restrict debugging and memory-dumping tools to trusted administrative accounts and apply the principle of least privilege.
### Sensitive Information Exposure via Process Memory
#### Description
A process named `/usr/bin/password-store`, running as `root`, retains the root account password in plaintext within its memory. The password can be recovered from a core dump of the process using `strings`.
#### Impact
An attacker who can obtain a memory dump of the process can retrieve the root password and use it to authenticate as `root`, resulting in complete system compromise.
#### Remediation
Avoid storing privileged credentials in plaintext within long-running processes. Minimize the time sensitive information remains in memory, clear sensitive buffers when no longer needed, and use secure credential management mechanisms. Rotate the exposed root password.