# Flight (HackTheBox) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show that DNS is running on port 53 (TCP), an Apache web server is running on port 80, Kerberos is running on port 88, LDAP is running on port 389 and SMB is running on port 445.
![](assets/flight/Pasted%20image%2020261117191612.png)

The web application on port 80 is a flight ticket booking website, though most of its features are non-functional.
![](assets/flight/Pasted%20image%2020261117191916.png)

The `flight.htb` hostname identified in the Nmap scan is added to the `/etc/hosts` file. The `ffuf` tool is then used to perform virtual host enumeration, revealing the `school.flight.htb` subdomain.
![](assets/flight/Pasted%20image%2020261117192044.png)

Enumeration of `http://school.flight.htb` shows that it utilizes a `view` parameter in the URL to render pages.
![](assets/flight/Pasted%20image%2020261117192109.png)

Initial attempts to test for Local File Inclusion (LFI) fail, with the application returning a `Suspicious Activity Blocked!` message.
![](assets/flight/Pasted%20image%2020261117192201.png)

Subsequent attempts to test for Remote File Inclusion (RFI) via web requests through the parameter do not trigger an error message from the application.
![](assets/flight/Pasted%20image%2020261117192342.png)

The `Responder` tool is configured in analysis mode to determine if the web application will initiate an outbound SMB request, which would allow capturing the NTLMv2 hash of the running user.
![](assets/flight/Pasted%20image%2020261117192306.png)

The `view` parameter is modified to reference a UNC path pointing to the attacking machine (`//10.10.15.99/hello`).
![](assets/flight/Pasted%20image%2020261117192519.png)

Responder successfully intercepts an incoming authentication request and captures the NTLMv2 hash belonging to the `svc_apache` service account.
![](assets/flight/Pasted%20image%2020261117192555.png)

The captured hash is successfully cracked offline using Hashcat.
![](assets/flight/Pasted%20image%2020261117192706.png)

Using the credentials of the low-privileged `svc_apache` user, domain user enumeration is performed, and the harvested usernames are saved to a file.
![](assets/flight/Pasted%20image%2020261117192808.png)

The `kerbrute` tool is used to perform a password spraying attack with `svc_apache`'s password, confirming that the user `S.Moon` shares the same password.
![](assets/flight/Pasted%20image%2020261117192908.png)

The newly obtained credentials are supplied to `bloodhound-python` to map domain relationships and analyze access control lists.
![](assets/flight/Pasted%20image%2020261117193114.png)

Additional NetExec enumeration indicates that `S.Moon` possesses `WRITE` permissions over the `Shared` SMB share.
![](assets/flight/Pasted%20image%2020261117193152.png)

NetExec's `slinky` module fails to generate a `.lnk` file pointing back to the attack host within the writable share.
![](assets/flight/Pasted%20image%2020261117193314.png)

The `ntlm-theft.py` script is used to generate multiple file formats pointing to the attacking machine, specifically selecting formats marked as `BROWSE TO FOLDER` to trigger automatic SMB requests when the directory is accessed via Windows Explorer.
![](assets/flight/Pasted%20image%2020261117193400.png)

The `htb.library-ms` and `desktop.ini` files are successfully uploaded to the target share.
![](assets/flight/Pasted%20image%2020261117193544.png)

Shortly after, Responder captures a new SMB authentication request from the user `C.Bum`.
![](assets/flight/Pasted%20image%2020261117193622.png)
## Initial Access
The NTLMv2 hash of the user is successfully cracked using Hashcat.
![](assets/flight/Pasted%20image%2020261117193653.png)

Further enumeration reveals that `C.Bum` holds `WRITE` permissions over the `Web` SMB share.
![](assets/flight/Pasted%20image%2020261117193723.png)

A PHP web shell is prepared to accept a `cmd` parameter via a GET request and execute its value on the underlying system using the `system()` function.
![](assets/flight/Pasted%20image%2020261117193817.png)

Using Impacket's `smbclient`, the web shell is uploaded directly to the root directory of the `http://flight.htb` web application.
![](assets/flight/Pasted%20image%2020261117193846.png)

Executing the `whoami` command through the uploaded web shell confirms execution under the context of `flight\svc_apache`.
![](assets/flight/Pasted%20image%2020261117193941.png)

A PowerShell reverse shell script (`powershell.ps1`) is configured to connect back to port 443 on the attacking machine.
![](assets/flight/Pasted%20image%2020261117194007.png)

A local Python web server is initiated using the `http.server` module to host and deliver the payload to the target.
![](assets/flight/Pasted%20image%2020261117194037.png)

The web shell executes a command utilizing PowerShell's `DownloadString` function to fetch and immediately execute the reverse shell script entirely in memory.
![](assets/flight/Pasted%20image%2020261117194145.png)

This establishes an interactive reverse shell session running as the `svc_apache` user.
![](assets/flight/Pasted%20image%2020261117194203.png)
## Privilege Escalation
The `Invoke-RunasCs.ps1` script is downloaded and transferred to the target system.
![](assets/flight/Pasted%20image%2020261117194237.png)

The script is executed with its built-in `Remote` parameter to spawn an authenticated reverse shell session in the context of the user `C.Bum`.
![](assets/flight/Pasted%20image%2020261117194300.png)

Executing `netstat` reveals that port 8000 on the target is currently in a `LISTENING` state.
![](assets/flight/Pasted%20image%2020261117194345.png)

Additionally, a `development` directory is located under `C:\inetpub\`, representing an application path separate from the XAMPP-hosted web applications.
![](assets/flight/Pasted%20image%2020261117194441.png)

A `chisel` binary is transferred to the target to establish a port-forwarding tunnel, mapping port 1337 on the attacker's machine to local port 8000 on the domain controller for direct browser access.
![](assets/flight/Pasted%20image%2020261117194529.png)

Navigating to the tunneled port displays a secondary flight ticket booking web application.
![](assets/flight/Pasted%20image%2020261117194630.png)

Using the Wappalyzer browser extension, it is identified that this application runs on the Microsoft ASP.NET framework rather than the PHP stack used by the primary site.
![](assets/flight/Pasted%20image%2020261117194749.png)

Further checks confirm that the user `c.bum` maintains write access to the development web directory, permitting file uploads.
![](assets/flight/Pasted%20image%2020261117194827.png)

Given the ASP.NET environment, an ASPX reverse shell payload is generated using `msfvenom`.
![](assets/flight/Pasted%20image%2020261117194908.png)

The payload is uploaded directly to the root directory of the development web application.
![](assets/flight/Pasted%20image%2020261117195004.png)

Browsing to the uploaded ASPX file executes the payload, establishing a reverse shell session as `iis apppool\defaultapppool`.
![](assets/flight/Pasted%20image%2020261117195051.png)

Running `whoami /all` confirms that this default service account possesses the `SeImpersonatePrivilege` user right.
![](assets/flight/Pasted%20image%2020261117195130.png)

The `SigmaPotato` tool is transferred to the target and executed to modify the password of the domain `Administrator` account.
![](assets/flight/Pasted%20image%2020261117195222.png)

Because the target acts as a Domain Controller, this action successfully updates the domain-wide `Administrator` password. Impacket's `psexec` is then used with the new credentials to authenticate to the DC as `NT AUTHORITY\SYSTEM`.
![](assets/flight/Pasted%20image%2020261117195255.png)
## Findings & Remediation

### Unvalidated Path Handling
#### Description
The web application improperly handles user-supplied input parameters, allowing an attacker to supply external UNC paths. 
#### Impact
An attacker can force the application to initiate unauthorized outbound SMB connections to an external system. This behavior can be leveraged to capture NetNTLMv2 authentication hashes from the service account running the application, which can subsequently be cracked offline to recover plaintext credentials.
#### Remediation
Input validation and sanitization mechanisms should be strictly implemented to ensure that parameters processing file or directory paths only accept local, whitelisted resources. Additionally, network perimeter firewalls and host-based policies should block outbound SMB traffic on port 445 to prevent internal servers from leaking authentication material to external hosts.
### Credential Reuse
#### Description
Domain user accounts utilize identical passwords across multiple services.
#### Impact
When a user account with reused credentials is compromised, attackers can leverage those same credentials to access additional resources and network shares. Write access to these shares enables attackers to plant malicious files, such as shortcut or library files, which automatically trigger outbound authentication requests and capture credentials from other corporate users interacting with the directory.
#### Remediation
Organizations should enforce robust password policies that strictly prohibit credential reuse across distinct user accounts and services. Furthermore, access control lists (ACLs) on network shares must follow the Principle of Least Privilege, restricting write access solely to authorized administrators to prevent the unauthorized placement of system-triggering files.
### Token Impersonation Privileges
#### Description
Service accounts and application pools are configured with default user rights such as `SeImpersonatePrivilege`.
#### Impact
An attacker who achieves code execution within a service account context possessing token impersonation privileges can leverage potato exploits. This enables local privilege escalation to higher-privileged accounts or system-level access, ultimately leading to total compromise of the affected machine.
#### Remediation
Remove unnecessary user rights, such as `SeImpersonatePrivilege` and `SeAssignPrimaryTokenPrivilege`, from service accounts and application pools where they are not strictly required. Run web applications and services under hardened, least-privileged service accounts rather than built-in defaults helps mitigate the risk of token manipulation and local escalation.