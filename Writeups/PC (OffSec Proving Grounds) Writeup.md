# PC (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal that SSH is running on port 22 in addition to two web servers. One is running on port 8000 and is identified as `ttyd (libwebsockets)`, with the `http-title` field showing `ttyd - Terminal`. The second is a Uvicorn web server running on port 65432.
![](assets/pc/Pasted%20image%2020261111173439.png)
## Initial Access
The web application on port 8000 provides a terminal where commands are executed directly on the target as the `user` user.
![](assets/pc/Pasted%20image%2020261111173702.png)
## Privilege Escalation
An `rpc.py` file is found in the `/opt` directory. Inspection of the file reveals that the Uvicorn service running on port 65432 hosts an RPC application.
![](assets/pc/Pasted%20image%2020261111173834.png)

The `ps aux` command shows that the service is running as `root`.
![](assets/pc/Pasted%20image%2020261111174004.png)

Searchsploit displays a remote code execution exploit for `rpc.py` version 0.6.0.
![](assets/pc/Pasted%20image%2020261111174129.png)

The proof of concept script reveals that this is a 0-day exploit for `rpc.py` with the vulnerability classified as CVE-2022-35411.
![](assets/pc/Pasted%20image%2020261111174207.png)

Since the process runs as `root`, commands executed through the exploit inherit root privileges. The exploit's main function is modified to copy `/bin/bash` to `/home/user/rootbash` and set the SUID bit on the copied binary.
![](assets/pc/Pasted%20image%2020261111174344.png)

The modified exploit is transferred to the target system and executed. The `rootbash` binary then appears in the user's home directory with the SUID bit set.
![](assets/pc/Pasted%20image%2020261111174603.png)

Executing `rootbash` with the `-p` parameter to maintain privileges spawns a root shell.
![](assets/pc/Pasted%20image%2020261111174703.png)
## Finding & Remediation
### Unauthenticated Web Terminal Access
#### Description
The target exposes a `ttyd` web terminal on port 8000 without requiring authentication. The terminal allows visitors to execute operating system commands directly as the `user` account.
#### Impact
An unauthenticated attacker can access the exposed terminal and execute arbitrary commands as the `user` account, gaining an initial foothold on the target system.
#### Remediation
Disable the web terminal if it is not required. Otherwise, enforce authentication and restrict access to trusted users and networks using firewall rules or an authenticated reverse proxy.
### Insecure Deserialization - CVE-2022-35411
#### Description
The target runs a vulnerable `rpc.py` service on port 65432. CVE-2022-35411 affects `rpc.py` versions through 0.6.0 and allows unauthenticated attackers to execute arbitrary code by triggering unsafe Python `pickle` deserialization through the `serializer: pickle` HTTP header.
#### Impact
An unauthenticated attacker can exploit this vulnerability to execute arbitrary commands with the privileges of the `root` account. In this case, the exploit was used to create a SUID-enabled copy of `/bin/bash`, allowing the attacker to obtain a root shell.
#### Remediation
Apply the upstream security fix for CVE-2022-35411, which prevents untrusted requests from triggering unsafe `pickle` deserialization. For installations that cannot be patched immediately, disable the `PickleSerializer` by removing it from `SERIALIZER_NAMES` and `SERIALIZER_TYPES`. Restrict access to the RPC service to trusted networks and avoid running it with `root` privileges.
