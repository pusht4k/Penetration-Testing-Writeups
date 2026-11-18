# Monitoring (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan reveals that the target is a Linux machine with SSH running on port 22, SMTP running on port 25, an Apache web server running on ports 80 and 443, and OpenLDAP running on port 389.
![](assets/monitoring/Pasted%20image%2020261110234456.png)

The web application's home page displays the Nagios XI welcome page.
![](assets/monitoring/Pasted%20image%2020261110234707.png)

The `creds` tool is used to search for default credentials for the Nagios XI application.
![](assets/monitoring/Pasted%20image%2020261110234738.png)

Using `nagiosadmin` for both the username and password fails. However, Burp Suite's Intruder is used to conduct a Sniper Attack using default passwords. This results in a redirection when the password `admin` is used, confirming successful authentication.
![](assets/monitoring/Pasted%20image%2020261110234925.png)

The obtained credentials `nagiosadmin:admin` are then used to access the application's dashboard. The application is identified as version 5.6.0.
![](assets/monitoring/Pasted%20image%2020261110235527.png)
## Initial Access
Searchsploit identifies a remote code execution vulnerability (CVE-2019-15949) affecting Nagios XI versions prior to 5.6.6, including the target's version 5.6.0.
![](assets/monitoring/Pasted%20image%2020261110235819.png)

The exploit is executed with the attacker's IP address and port 443 specified for the reverse shell connection.
![](assets/monitoring/Pasted%20image%2020261110235846.png)

This results in a reverse shell as the `root` user.
![](assets/monitoring/Pasted%20image%2020261110235946.png)
## Privilege Escalation
Privilege escalation is not required for this lab as the initial exploit compromises the root account immediately.
## Finding & Remediation
### Use of Weak Credentials
#### Description
The Nagios XI application uses a weak password (`admin`) for the `nagiosadmin` account, allowing the credentials to be identified through a password guessing attack.
#### Impact
An unauthenticated attacker can obtain administrative access to the Nagios XI application by guessing the weak password.
#### Remediation
Enforce a strong password policy and replace weak passwords with unique, complex passwords. Implement account lockout or rate-limiting mechanisms to mitigate password guessing attacks.
### Nagios XI Remote Code Execution (CVE-2019-15949)
#### Description
The target runs Nagios XI version 5.6.0, which is vulnerable to CVE-2019-15949. This vulnerability allows authenticated attackers to execute arbitrary commands on the underlying system.
#### Impact
An attacker with valid administrative credentials can exploit this vulnerability to execute arbitrary commands as the `root` user, resulting in complete system compromise.
#### Remediation
Upgrade Nagios XI to version 5.6.6 or later to address CVE-2019-15949. Restrict access to the Nagios XI administrative interface to trusted users and networks.
