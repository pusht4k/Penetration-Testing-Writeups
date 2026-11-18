# Fluffy (HackTheBox) Writeup

## Enumeration
This lab simulates a breach scenario, providing credentials for a low-privileged user:
```
Username: j.fleischman
Password: J0elTHEM4n1990!
```
#### Nmap Scan Results
An Nmap scan reveals multiple services on the target. The most notable ones are Kerberos (port 88), LDAP (port 389), SMB (port 445), and WinRM (port 5985).
![](assets/fluffy/Pasted%20image%2020261117201253.png)

Using the provided credentials to connect via SMB, the `IT` share is accessed, revealing both `READ` and `WRITE` permissions. Inside, a file named `Upgrade_Notice.pdf` is found and downloaded to the attack host.
![](assets/fluffy/Pasted%20image%2020261117201540.png)

Reviewing the PDF reveals details of a recent vulnerability assessment that mentions several CVE numbers.
![](assets/fluffy/Pasted%20image%2020261117201615.png)

Researching one of the mentioned CVEs leads to the NIST page for `CVE-2025-24071`, a Windows Explorer spoofing vulnerability that can be leveraged to leak sensitive information.
![](assets/fluffy/Pasted%20image%2020261117201710.png)

Further research points to a public proof of concept on [GitHub (by LOOKY243)](https://github.com/LOOKY243/CVE-2025-24071-PoC) that generates a `.library-ms` file inside a ZIP archive. When extracted or viewed, it triggers an outbound SMB authentication request, allowing the NTLMv2 hash of any user interacting with the archive to be captured.
![](assets/fluffy/Pasted%20image%2020261117201834.png)

The exploit is used to generate the malicious file, pointing it back to the attack host, and it is uploaded to the `IT` SMB share.
![](assets/fluffy/Pasted%20image%2020261117202000.png)

A short time later, Responder successfully captures the NTLMv2 hash belonging to `FLUFFY\p.agila`.
![](assets/fluffy/Pasted%20image%2020261117202048.png)

The hash is cracked offline using Hashcat. Although share permissions are checked using NetExec with the new credentials, `p.agila` shares the same access level as `j.fleischman`.
![](assets/fluffy/Pasted%20image%2020261117202110.png)

Next, `bloodhound-python` is run to map domain relationships and Access Control Lists (ACLs).
![](assets/fluffy/Pasted%20image%2020261117202224.png)

BloodHound reveals that `p.agila` is a member of the `SERVICE ACCOUNT MANAGERS` group, which holds `GenericAll` privileges over the `SERVICE ACCOUNTS` group. In turn, the `SERVICE ACCOUNTS` group possesses `GenericWrite` privileges over three service accounts: `winrm_svc`, `ca_svc`, and `ldap_svc`.
![](assets/fluffy/Pasted%20image%2020261117202353.png)

The `winrm_svc` account is also a member of the `REMOTE MANAGEMENT USERS` group, enabling WinRM access to the Domain Controller.
![](assets/fluffy/Pasted%20image%2020261117202516.png)

Because of the `GenericAll` permission, `p.agila` can be added directly to the `SERVICE ACCOUNTS` group, inheriting `GenericWrite` access over the service accounts. This permission can then be leveraged to perform a Shadow Credentials attack using Certipy to extract the account's NT hash.

First, the user is added to the `SERVICE ACCOUNTS` group using Samba's `net` utility.
![](assets/fluffy/Pasted%20image%2020261117202749.png)

Certipy is then used to execute a Shadow Credentials attack against `winrm_svc`. The attack succeeds, and Certipy extracts the NT hash of the service account.
![](assets/fluffy/Pasted%20image%2020261117202854.png)

Using this hash, a Pass-the-Hash attack is performed via `evil-winrm` to log into the Domain Controller and retrieve the user flag from `winrm_svc`'s Desktop.
![](assets/fluffy/Pasted%20image%2020261117202922.png)
## Privilege Escalation
While the current service account permissions do not immediately provide a path to domain admin, the presence of `ca_svc` indicates an active Certificate Authority (AD CS), which is confirmed using NetExec's `adcs` module.
![](assets/fluffy/Pasted%20image%2020261117203028.png)

Another Shadow Credentials attack is performed against `ca_svc` to obtain its NT hash.
![](assets/fluffy/Pasted%20image%2020261117203102.png)

Using Certipy to enumerate the Certificate Authority reveals that it is vulnerable to **ESC16**.
ESC16 occurs when the AD CS environment is misconfigured to use weak certificate mapping methods (such as relying on UPN mapping without enforcing strict certificate mapping rules). If an account has permissions to modify its own `userPrincipalName` (UPN), it can be temporarily changed to `Administrator`, a certificate requested, and the UPN change reverted. When authenticating, the CA maps the resulting certificate to the `Administrator` account due to the weak mapping configuration.
![](assets/fluffy/Pasted%20image%2020261117203327.png)

This vulnerability is abused through the following steps:
First, the current `userPrincipalName` of `ca_svc` is retrieved.
![](assets/fluffy/Pasted%20image%2020261117203836.png)

The attribute is modified to `Administrator`.
![](assets/fluffy/Pasted%20image%2020261117203906.png)

A certificate is requested using the standard `User` template. Because of the temporary UPN, the issued certificate is linked to the `Administrator` account.
![](assets/fluffy/Pasted%20image%2020261117203945.png)

The `userPrincipalName` of `ca_svc` is immediately reverted to its original value. The obtained certificate is then used to authenticate via Kerberos and request a TGT, allowing Certipy to extract the Administrator's NTLM hash.
![](assets/fluffy/Pasted%20image%2020261117204044.png)

Finally, the extracted hash is used in a Pass-the-Hash attack via `evil-winrm` to authenticate as `Administrator` and retrieve the root flag from `C:\Users\Administrator\Desktop\`.
![](assets/fluffy/Pasted%20image%2020261117204128.png)
## Findings & Remediation

### Windows Spoofing (CVE-2025-24071)
#### Description
The environment is affected by an unpatched vulnerability in Windows Explorer (tracked as CVE-2025-24071), which can be leveraged through manipulated file formats stored on accessible network shares.
#### Impact
An attacker with write access to an internal file share can plant malicious files, such as custom library or shortcut files, which automatically trigger outbound SMB authentication requests when accessed or viewed. This allows attackers to capture NetNTLMv2 hashes from domain users and service accounts interacting with the share, leading to offline password cracking and credential theft.
#### Remediation
Organizations should promptly apply the latest security patches and updates released by Microsoft to address CVE-2025-24071 and related Windows Explorer spoofing vulnerabilities. Additionally, network perimeter firewalls and group policies should restrict outbound SMB traffic to prevent internal servers and workstations from leaking authentication material to external hosts.
### Excessive ACL Configuration
#### Description
Active Directory objects and security groups are configured with overly permissive Access Control Lists (ACLs), such as `GenericAll` and `GenericWrite` permissions assigned to non-privileged security groups over service accounts.
#### Impact
Excessive ACL privileges enable low-privileged users or compromised accounts to manipulate critical directory objects, modify group memberships, or execute Shadow Credentials attacks against service accounts. This allows attackers to escalate privileges within the Active Directory environment and compromise secondary service accounts required for further lateral movement.
#### Remediation
Administrators should strictly enforce the Principle of Least Privilege by regularly auditing Active Directory Access Control Lists (ACLs) and removing unauthorized permissions like `GenericAll` and `GenericWrite` from standard user accounts and security groups. Permissions over service accounts and sensitive domain objects must be restricted solely to authorized management groups.
### Vulnerable Certificate Authority (ESC16)
#### Description
The Active Directory Certificate Services (AD CS) environment is misconfigured to utilize weak certificate mapping methods, combined with permissions allowing accounts to modify their own `userPrincipalName` (UPN) attribute.
#### Impact
An attacker who gains control over an account capable of modifying its own UPN attribute can temporarily alter it to match high-privileged entities, such as the `Administrator` account. By requesting a certificate under this temporary UPN and reverting the change, the attacker can leverage the weak certificate mapping configuration to authenticate as the elevated account, leading to complete domain compromise.
#### Remediation
Certificate Authorities must be hardened by enforcing strict certificate mapping rules that prohibit weak UPN or alternative name fallbacks. Furthermore, object permissions within Active Directory must be reviewed to ensure standard users cannot modify sensitive attributes such as `userPrincipalName` on accounts that do not belong to them.
