# Nibbles (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show FTP running on port 21, SSH running on port 22, an Apache web server running on port 80 and PostgreSQL running on port 5437.
![](assets/nibbles/Pasted%20image%2020261111183432.png)

Enumeration of the web application does not lead to exploitation, as the web server only contains two mostly empty pages and a static image at `/nightmare.jpg`.
![](assets/nibbles/Pasted%20image%2020261111183523.png)

The PostgreSQL service is configured with default credentials, as `postgres` is used for both the username and the password.
![](assets/nibbles/Pasted%20image%2020261111183646.png)
## Initial Access
Enumeration of PostgreSQL reveals that the databases are empty. However, command execution is possible through PostgreSQL using the `COPY ... FROM PROGRAM` statement. This is exploited to run a Bash reverse shell one-liner that connects back to port 443 on the attacking machine.
![](assets/nibbles/Pasted%20image%2020261111183837.png)

The command is executed directly on the target system, establishing a reverse shell as the `postgres` user.
![](assets/nibbles/Pasted%20image%2020261111183930.png)
## Privilege Escalation
The `find` command is used to search for all files on the target with the SUID or SGID bits set, revealing that `/usr/bin/find` is configured with the SUID bit.
![](assets/nibbles/Pasted%20image%2020261111184017.png)

This is exploited to execute `/bin/bash` with elevated privileges, gaining access to the `root` account.
![](assets/nibbles/Pasted%20image%2020261111184132.png)

## Finding & Remediation
### Use of Default Credentials
#### Description
The PostgreSQL service is configured with the default credentials `postgres:postgres`, allowing unauthenticated users to access the database using publicly known credentials.
#### Impact
An attacker can use the default credentials to authenticate to PostgreSQL with administrative privileges and access the database.
#### Remediation
Replace the default PostgreSQL password with a strong, unique password. Restrict database access to trusted hosts and enforce appropriate authentication controls.
### PostgreSQL Command Execution
#### Description
The PostgreSQL `postgres` account has sufficient privileges to execute operating system commands through the `COPY ... FROM PROGRAM` statement.
#### Impact
An attacker with access to the PostgreSQL administrative account can execute arbitrary commands on the underlying system and establish a reverse shell as the `postgres` user.
#### Remediation
Restrict the use of `COPY ... FROM PROGRAM` to trusted administrative roles by removing unnecessary superuser privileges and membership in the `pg_execute_server_program` role. Apply the principle of least privilege to database accounts and avoid using administrative accounts for routine operations.
### Misconfigured SUID Bit
#### Description
The `/usr/bin/find` binary is configured with the SUID bit, allowing it to execute with the privileges of its owner (`root`). This configuration enables lower-privileged users to execute commands with elevated privileges.
#### Impact
An attacker with access to any local user account can exploit the SUID-enabled `find` binary to execute `/bin/bash` with `root` privileges, resulting in complete system compromise.
#### Remediation
Remove the unnecessary SUID bit from `/usr/bin/find` using `sudo chmod u-s /usr/bin/find`. Audit other SUID and SGID binaries to ensure that elevated permissions are granted only when required.
