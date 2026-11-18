# Forest (HackTheBox) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show notable services commonly associated with Windows Active Directory domain controllers, such as Kerberos (port 88), LDAP (port 389), SMB (port 445), and WinRM (port 5985). The scan results also display the domain name, `htb.local`.
![](assets/forest/Pasted%20image%2020261117205359.png)

Null LDAP authentication is allowed, revealing the domain's Distinguished Name (DN): `DC=htb,DC=local`.
![](assets/forest/Pasted%20image%2020261117205544.png)

Additional LDAP enumeration reveals the `svc-alfresco` service account.
![](assets/forest/Pasted%20image%2020261117205643.png)

This account is a well-known Active Directory service account associated with the Hyland Alfresco Enterprise Content Management platform. Due to Kerberos misconfigurations, the account does not require pre-authentication, exposing it to an AS-REP Roasting attack. Impacket's `GetNPUsers` script is used to request and obtain the AS-REP ticket from Kerberos.
![](assets/forest/Pasted%20image%2020261117210004.png)
## Initial Access
Hashcat successfully cracks the hash and recovers the user's plaintext password: `s3rvice`.
![](assets/forest/Pasted%20image%2020261117210044.png)

NetExec confirms that the `svc-alfresco` account can authenticate to the target through WinRM.
![](assets/forest/Pasted%20image%2020261117210100.png)

The `evil-winrm` tool is used to access the domain controller and retrieve the user flag from `svc-alfresco`'s Desktop.
![](assets/forest/Pasted%20image%2020261117210149.png)
## Privilege Escalation
BloodHound enumeration reveals that `svc-alfresco` is a member of nine groups through nested memberships, including the `ACCOUNT OPERATORS` group.
![](assets/forest/Pasted%20image%2020261117210806.png)

The `ACCOUNT OPERATORS` group holds `GenericAll` permissions over the `EXCHANGE WINDOWS PERMISSIONS` domain group.
![](assets/forest/Pasted%20image%2020261117211249.png)

In turn, the `EXCHANGE WINDOWS PERMISSIONS` group possesses `WriteDACL` permissions over the domain.
![](assets/forest/Pasted%20image%2020261117211405.png)

These chained privileges allow creating a new user and adding it to the `EXCHANGE WINDOWS PERMISSIONS` group. The new account can then be granted `DCSync` privileges to extract NTLM hashes for all domain users from the Domain Controller.
First, a new user (`pusht4k`) is created and added to the `EXCHANGE WINDOWS PERMISSIONS` group.
![](assets/forest/Pasted%20image%2020261117211551.png)

Impacket's `dacledit` is used to grant the `pusht4k` account `DCSync` permissions over the domain.
![](assets/forest/Pasted%20image%2020261117211611.png)

Impacket's `secretsdump` is then used to perform a DCSync attack and retrieve the NTLM hash of the domain `Administrator` account.
![](assets/forest/Pasted%20image%2020261117211657.png)

Finally, the extracted hash is used in a Pass-the-Hash attack via Impacket's `psexec` tool to authenticate to the DC as `NT AUTHORITY\SYSTEM`. The root flag is retrieved from the `Administrator` account's Desktop.
![](assets/forest/Pasted%20image%2020261117211742.png)
## Findings & Remediation

### Unauthenticated LDAP Access

#### Description
The Active Directory environment is configured to permit unauthenticated (null) LDAP binds, allowing anonymous clients to query the directory service.
#### Impact
An unauthenticated attacker can query the Active Directory database to harvest domain structures, usernames, computer accounts, and specific service accounts. This allows an external or internal actor to gather valuable reconnaissance data and identify vulnerable high-value targets without requiring any valid domain credentials.
#### Remediation
Unauthenticated LDAP binding should be disabled across all domain controllers by modifying RootDSE security restrictions and enforcing strict authentication requirements for directory queries.
### Kerberos Pre-Authentication Disabled (AS-REP Roasting)
#### Description
A domain service account is configured with the `Do not require Kerberos preauthentication` flag enabled, and its associated password is weak and susceptible to offline brute-forcing.
#### Impact
An attacker can request an authentication ticket for the account without providing valid credentials and subsequently crack the ticket offline using brute-force dictionaries. Because the account utilizes a weak password, this allows attackers to successfully recover the plaintext credentials, authenticate to the domain via WinRM and achieve initial access.
#### Remediation
Kerberos pre-authentication must be enforced for all domain user accounts, especially service accounts, alongside the implementation of robust, complex password policies that prevent the use of easily crackable passwords.
### Misconfigured Domain ACL Privileges
#### Description
Active Directory permissions are improperly configured through nested groups and chained access control lists (ACLs). Through nested membership in the `ACCOUNT OPERATORS` group, the `svc-alfresco` service account can add users to the `EXCHANGE WINDOWS PERMISSIONS` group, which has the `WriteDACL` privilege over the domain.
#### Impact
These ACLs allow an attacker with access to the `svc-alfresco` service account to create a new domain user and add it to the `EXCHANGE WINDOWS PERMISSIONS` group. The user then inherits the `WriteDACL` privilege over the domain, allowing it to grant itself the DCSync privilege, leading to full domain compromise.
#### Remediation
Active Directory Access Control Lists (ACLs) should be regularly audited and restricted to enforce the Principle of Least Privilege, ensuring that sensitive permissions are removed from the `svc-alfresco` account and are only granted to highly privileged users such as system administrators.
