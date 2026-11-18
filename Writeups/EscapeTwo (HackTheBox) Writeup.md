# EscapeTwo (HackTheBox) Writeup

## Enumeration
This lab simulates a breach scenario, providing the following credentials for a low-privileged user:
```
Username: rose
Password: KxEPkKe6R8su
```
#### Nmap Scan Results
The Nmap scan results show services commonly associated with Windows Active Directory environments, with DNS running on TCP port 53, Kerberos running on port 88, LDAP running on ports 389 and 636, SMB running on port 445, MSSQL on port 1433, and WinRM available on port 5985.
```
Nmap scan report for 10.129.71.221
Host is up (0.070s latency).
Not shown: 65511 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-01-13 13:04:17Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
<SNIP>
```

The `smbmap` tool is used to enumerate accessible shares in SMB. The output shows that the provided user can access the `Accounting Department` share.
![](assets/escapetwo/Pasted%20image%2020261117182250.png)

The share contains two Microsoft Excel files, which are downloaded to the attacking machine.
![](assets/escapetwo/Pasted%20image%2020261117182345.png)

The `file` command reveals that the `accounts.xlsx` file is actually a ZIP archive rather than a true Excel file.
![](assets/escapetwo/Pasted%20image%2020261117182726.png)

The files are extracted from the archive using the `unzip` command. An `xl` directory is also extracted, containing the `sharedStrings.xml` file.
![](assets/escapetwo/Pasted%20image%2020261117182807.png)

The file is reviewed, revealing usernames alongside an associated password for each user.
![](assets/escapetwo/Pasted%20image%2020261117183003.png)

NetExec is used to test the credentials against the domain controller. The tool confirms that the credentials for the `oscar` account are valid.
![](assets/escapetwo/Pasted%20image%2020261117183106.png)

The obtained credentials are used to enumerate domain users through NetExec.
![](assets/escapetwo/Pasted%20image%2020261117183631.png)

The `sa` user's credentials found in the XML file are also valid and are used to authenticate to the MSSQL service through Impacket's `mssqlclient` tool. The user context is shown as `dbo`, confirming database owner privileges.
![](assets/escapetwo/Pasted%20image%2020261117183257.png)

The `xp_cmdshell` function is then enabled and used to execute the `whoami` command. Commands execute in the context of `sequel\sqv_svc`.
![](assets/escapetwo/Pasted%20image%2020261117183421.png)

An SQL configuration file is found in `C:\SQL2019\ExpressAdv_ENU\`, containing a password.
![](assets/escapetwo/Pasted%20image%2020261117183539.png)
## Initial Access
This password is sprayed against the domain users discovered earlier. NetExec confirms that the password belongs to the `ryan` user.
![](assets/escapetwo/Pasted%20image%2020261117183743.png)

Further enumeration shows that the `ryan` user can access the domain controller via the WinRM protocol.
![](assets/escapetwo/Pasted%20image%2020261117183851.png)

The `evil-winrm` tool is used to gain access to the target. The user flag is found on `ryan`'s Desktop.
![](assets/escapetwo/Pasted%20image%2020261117183943.png)
## Privilege Escalation
The `bloodhound-python` tool is used to enumerate domain relationships and ACLs.
![](assets/escapetwo/Pasted%20image%2020261117184022.png)

BloodHound shows that `ryan` has the `WriteOwner` privilege over `ca_svc`.
![](assets/escapetwo/Pasted%20image%2020261117184112.png)

This privilege can be used to set `ryan` as the owner of `ca_svc`, and subsequently grant `GenericAll` permissions. This allows modifying account attributes to perform attacks such as Kerberoasting or Shadow Credentials, or directly forcing a password change.

Impacket's `owneredit` script is used to set `ryan` as the new owner of the `ca_svc` user.
![](assets/escapetwo/Pasted%20image%2020261117184319.png)

Impacket's `dacledit` is then used to grant `ryan` `FullControl` (`GenericAll`) over the service account.
![](assets/escapetwo/Pasted%20image%2020261117184422.png)

Certipy-AD is used to perform a Shadow Credentials attack. The tool executes successfully and extracts the NT hash of the service account.
![](assets/escapetwo/Pasted%20image%2020261117184456.png)

Certipy is then used to enumerate vulnerable templates in the Certificate Authority. The tool indicates that the `DunderMifflinAuthentication` template is vulnerable to `ESC4`.
![](assets/escapetwo/Pasted%20image%2020261117184614.png)

The official Certipy documentation provides further details on this vulnerability, which allows modifying template configurations to introduce other vulnerabilities, such as `ESC1`, enabling user impersonation.
![](assets/escapetwo/Pasted%20image%2020261117184701.png)

The tool is then used to modify the template using the `-write-default-configuration` parameter.
![](assets/escapetwo/Pasted%20image%2020261117184939.png)

With the `DunderMifflinAuthentication` template now vulnerable to ESC1, a certificate is successfully requested for the domain `Administrator` account.
![](assets/escapetwo/Pasted%20image%2020261117185029.png)

The obtained certificate is used for Kerberos authentication to obtain a TGT, allowing Certipy to extract the user's NTLM hash.
![](assets/escapetwo/Pasted%20image%2020261117185131.png)

The hash is used in a Pass-the-Hash attack via `evil-winrm` to authenticate to the DC as `Administrator`. The root flag is found on the Administrator's desktop.
![](assets/escapetwo/Pasted%20image%2020261117185153.png)
## Findings & Remediation

### Sensitive Information Disclosure
#### Description
The `Accounting Department` SMB share contains files that expose plain-text usernames and associated passwords.  Additionally, the SQL configuration file reveals a plain-text password for the `sql_svc` account, that is later confirmed to be used for another domain user.
#### Impact
An attacker with access to low-privileged network shares can retrieve plain-text credentials, allowing them to authenticate to other network services, perform lateral movement, or gain unauthorized access to internal resources.
#### Remediation
Restrict permissions on SMB shares to ensure only authorized users or groups can access sensitive directories. Additionally, conduct regular audits of shared resources to remove files containing hardcoded or plain-text credentials.
### Password Reuse
#### Description
A plain-text password for the `sql_svc` account was discovered within an SQL configuration file on the local file system. This password was subsequently found to be reused by a domain user account (`ryan`).
#### Impact
Account and password reuse further expands the attack surface, enabling attackers to leverage a single compromised credential to access multiple services or accounts across the domain.
#### Remediation
Enforce strict password policies that prohibit credential reuse across multiple user accounts and services.
### Excessive ACL Privileges
#### Description
A standard domain user (`ryan`) was assigned excessive control rights (`WriteOwner`) over a service account (`ca_svc`).
#### Impact
Excessive ACL privileges allow low-privileged users to take ownership of sensitive objects or grant themselves full control permissions (`GenericAll`). An attacker can exploit this to manipulate account attributes, force password changes, perform attacks like Shadow Credentials, or compromise critical service accounts to escalate privileges within the Active Directory environment.
#### Remediation
Implement the Principle of Least Privilege (PoLP) by reviewing and stripping unnecessary ACL permissions granted to standard users over sensitive domain objects and service accounts. Regularly audit Active Directory Access Control Lists (ACLs) to detect and remediate dangerous permission assignments such as `WriteOwner` and `GenericAll`.
### Vulnerable Certificate Service (ESC4)
#### Description
A Certificate Authority (CA) template (`DunderMifflinAuthentication`) suffered from ESC4, allowing modification of its configuration. This misconfiguration enabled the template to be altered into a state vulnerable to ESC1, permitting the issuance of certificates on behalf of other users, including high-privileged accounts.
#### Impact
An attacker with modification rights over a certificate template can reconfigure it to allow subject alternative name (SAN) specification (ESC1). This enables the attacker to request a valid certificate impersonating any domain user, including the `Administrator` account, leading to complete domain compromise and full control over the Domain Controller.
#### Remediation
Secure Certificate Authority (CA) templates by restricting access control lists (ACLs) so that only authorized administrators can modify template configurations, mitigating ESC4 risks. Enforce secure template settings, such as disabling Manager Approval bypass, restricting enrollment permissions, and preventing low-privileged users from specifying arbitrary Subject Alternative Names (SANs) to protect against ESC1.
