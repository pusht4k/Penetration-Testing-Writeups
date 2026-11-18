# Escape (HackTheBox) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan reveals several open ports, including DNS on port 53, Kerberos on port 88, LDAP on port 389, SMB on port 445, LDAPS on port 636 and MSSQL on port 1433.
The presence of Active Directory services, along with the DNS name `dc.sequel.htb` identified in the LDAP SSL certificate, indicates that the host is a Domain Controller.
![](assets/escape/Pasted%20image%2020261117174316.png)

The `smbmap` tool displays an accessible `Public` share when connecting to the SMB service using null authentication.
![](assets/escape/Pasted%20image%2020261117174650.png)

A PDF file stored inside the share is downloaded to the attacking machine.
![](assets/escape/Pasted%20image%2020261117174827.png)

The file contains the names of two employees, which are noted as potential usernames.
![](assets/escape/Pasted%20image%2020261117174951.png)

The PDF also reveals credentials for the MSSQL database.
![](assets/escape/Pasted%20image%2020261117175024.png)

These credentials are used to authenticate to MSSQL using Impacket's `mssqlclient` tool. While the connection is displayed as `guest`, the `enable_xp_cmdshell` command is used to test whether guest users are configured incorrectly. The server blocks this as the user does not have the required permissions.
![](assets/escape/Pasted%20image%2020261117175121.png)

However, the `xp_dirtree` command is not blocked. The command is used to point to a non-existent file on the attacking machine in order to extract the NTLMv2 hash of the user running the MSSQL server through Responder.
![](assets/escape/Pasted%20image%2020261117175229.png)

Responder is used in analyze mode (`-A` parameter) and successfully captures the NTLMv2 hash of the `sql_svc` user.
![](assets/escape/Pasted%20image%2020261117175404.png)
## Initial Access
Offline hash cracking with Hashcat is successful, revealing the `sql_svc` account's password: `REGGIE1234ronnie`.
![](assets/escape/Pasted%20image%2020261117175431.png)

NetExec is used to test authentication through WinRM. The output reveals that the user is allowed to access the domain controller directly using this protocol. The `evil-winrm` tool is then used to authenticate to the DC as `sql_svc`.
![](assets/escape/Pasted%20image%2020261117175526.png)
## Privilege Escalation
An `ERRORLOG.BAK` file is found in `C:\SQLServer\Logs\`.
![](assets/escape/Pasted%20image%2020261117175712.png)

The log file contains two failed login attempts. The first one is for the `Ryan.Cooper` account, while the second one appears to contain a password entered instead of a username.
![](assets/escape/Pasted%20image%2020261117175737.png)

NetExec is used once again to test the user's credentials, revealing that the credentials are correct and `Ryan.Cooper` can also use WinRM to authenticate to the DC.
![](assets/escape/Pasted%20image%2020261117175832.png)

The user flag is found on `Ryan.Cooper`'s Desktop.
![](assets/escape/Pasted%20image%2020261117180734.png)

The `whoami /all` command is used to display information about the `Ryan.Cooper` account. The output shows that the user is a member of the `BUILTIN\Certificate Service DCOM Access` group.
![](assets/escape/Pasted%20image%2020261117175956.png)

Certipy-AD is then used to enumerate Active Directory Certificate Services (AD CS). The tool identifies a certificate template vulnerable to ESC1.
```
└─# certipy-ad find -vulnerable -stdout -u ryan.cooper@sequel.htb -p 'NuclearMosquito3' -dc-ip 10.129.63.5 
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Finding issuance policies
[*] Found 15 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'sequel-DC-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'sequel-DC-CA'
[*] Checking web enrollment for CA 'sequel-DC-CA' @ 'dc.sequel.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : sequel-DC-CA
    DNS Name                            : dc.sequel.htb
    Certificate Subject                 : CN=sequel-DC-CA, DC=sequel, DC=htb
    Certificate Serial Number           : 1EF2FA9A7E6EADAD4F5382F4CE283101
    Certificate Validity Start          : 2022-11-18 20:58:46+00:00
    Certificate Validity End            : 2121-11-18 21:08:46+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : SEQUEL.HTB\Administrators
      Access Rights
        ManageCa                        : SEQUEL.HTB\Administrators
                                          SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        ManageCertificates              : SEQUEL.HTB\Administrators
                                          SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        Enroll                          : SEQUEL.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : UserAuthentication
    Display Name                        : UserAuthentication
    Certificate Authorities             : sequel-DC-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Client Authentication
                                          Secure Email
                                          Encrypting File System
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 10 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2022-11-18T21:10:22+00:00
    Template Last Modified              : 2024-01-19T00:26:38+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Domain Users
                                          SEQUEL.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : SEQUEL.HTB\Administrator
        Full Control Principals         : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        Write Owner Principals          : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        Write Dacl Principals           : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        Write Property Enroll           : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Domain Users
                                          SEQUEL.HTB\Enterprise Admins
    [+] User Enrollable Principals      : SEQUEL.HTB\Domain Users
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```

The Certipy wiki provides more information about the vulnerability, which allows low-privileged users to request certificates on behalf of other domain accounts through a misconfigured certificate template. This can lead to unauthorized impersonation of privileged domain accounts, including domain administrators.
![](assets/escape/Pasted%20image%2020261117180153.png)

Certipy-AD is then used to request a certificate for the `Administrator` user, specifying the vulnerable template and the `Administrator`'s SID.
![](assets/escape/Pasted%20image%2020261117180501.png)

The obtained certificate is used to authenticate to Kerberos and request a TGT. The `faketime` command is used to avoid the Kerberos CLOCK SKEW error. The Certipy tool extracts the NTLM hash for the domain `Administrator` account.
![](assets/escape/Pasted%20image%2020261117180555.png)

The NTLM hash is then used in a Pass-the-Hash attack through Impacket's `psexec` script to authenticate to the DC as `NT AUTHORITY\SYSTEM` and retrieve the root flag from `C:\Users\Administrator\Desktop`.
![](assets/escape/Pasted%20image%2020261117180705.png)
## Finding & Remediation
### Unauthenticated SMB Access
#### Description
The SMB service allows unauthenticated access to the `Public` share, which contains internal documentation and MSSQL credentials.
#### Impact
An unauthenticated attacker can access sensitive information without providing valid credentials. In this case, the exposed documentation revealed employee names and database credentials.
#### Remediation
Disable anonymous SMB access and restrict share permissions to authorized users. Remove sensitive credentials from shared documentation and rotate the exposed credentials.
### Insecure MSSQL Configuration
#### Description
The MSSQL server allows low-privileged database users to access functionality that can initiate outbound network authentication using the MSSQL service account.
#### Impact
This configuration exposes the service account's authentication material to unauthorized parties. In this case, the exposed authentication material resulted in the compromise of the `sql_svc` account.
#### Remediation
Restrict access to unnecessary extended stored procedures, block unauthorized outbound SMB connections and configure the MSSQL service to use a dedicated least-privileged service account with a strong, randomly generated password. Consider using a Group Managed Service Account (gMSA).
### Sensitive Information Disclosure in Log Files
#### Description
The MSSQL backup log file contains sensitive authentication information resulting from a password being entered into the username field during a failed login attempt.
#### Impact
Users with access to the log file can obtain sensitive credentials. In this case, the exposed password resulted in the compromise of the `Ryan.Cooper` domain account.
#### Remediation
Restrict access to database log files, implement appropriate log retention and sanitization policies and immediately rotate exposed credentials. Ensure that authentication logs do not unnecessarily expose sensitive user input.
### Misconfigured AD CS Certificate Template (ESC1)
#### Description
The `UserAuthentication` certificate template is configured with insecure enrollment permissions and subject configuration, allowing unauthorized identity impersonation through certificates intended for client authentication.
#### Impact
The misconfigured certificate template allows low-privileged domain users to impersonate privileged accounts. In this case, exploitation resulted in the compromise of the domain Administrator account and full control over the domain controller.
#### Remediation
Disable the ability for enrollees to supply arbitrary subject information, restrict certificate enrollment permissions to authorized principals and remove unnecessary client authentication capabilities from certificate templates. Implement certificate approval requirements where appropriate and regularly audit AD CS configurations.
