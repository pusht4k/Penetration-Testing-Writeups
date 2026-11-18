
# SoSimple (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
An Nmap scan reveals SSH running on port 22 and an Apache web server running on port 80.
![](assets/sosimple/Pasted%20image%2020261110000326.png)

A `gobuster` scan reveals a WordPress installation at the `/wordpress` endpoint.
![](assets/sosimple/Pasted%20image%2020261110000429.png)

WPScan is used to enumerate the application, revealing that the Social Warfare plugin is installed and is vulnerable to an unauthenticated remote code execution exploit (CVE-2019-9978).
![](assets/sosimple/Pasted%20image%2020261110000508.png)

## Initial Access
A proof-of-concept Python script is found using Searchsploit.
![](assets/sosimple/Pasted%20image%2020261110000812.png)

The exploit is modified to point to port 443 on the attacking machine.
![](assets/sosimple/Pasted%20image%2020261110000859.png)

The exploit is then successfully executed, establishing a reverse shell running as the `www-data` user.
![](assets/sosimple/Pasted%20image%2020261110000937.png)

The user flag is found in the `max` user's home directory.
![](assets/sosimple/Pasted%20image%2020261110002205.png)

## Privilege Escalation
A private SSH key is found in the `max` user's SSH directory, with all users on the system allowed to read the file.
![](assets/sosimple/Pasted%20image%2020261110001106.png)

The key is copied to the attacking machine, given proper permissions, and used to access the host as `max`.
![](assets/sosimple/Pasted%20image%2020261110001207.png)

Using the `sudo -l` command reveals that `max` can execute `/usr/bin/service` as the user `steven` using `sudo` without providing a password.
![](assets/sosimple/Pasted%20image%2020261110001313.png)

This is used to execute `/bin/bash` using a relative path to spawn a shell running as `steven`.
![](assets/sosimple/Pasted%20image%2020261110001420.png)

A subsequent use of `sudo -l` as `steven` shows that the user can execute `/opt/tools/server-health.sh` as `root` with no password requirement.
![](assets/sosimple/Pasted%20image%2020261110001533.png)

Inspection of the `/opt` directory reveals that it is empty. However, `steven` is the owner of the directory and has write permissions on it. This allows abuse of the sudo misconfiguration by creating a malicious script at the specified path that would be executed with root privileges.

The `tools` directory is then created and a `server-health.sh` script containing a Bash reverse shell one-liner is stored in the directory.
![](assets/sosimple/Pasted%20image%2020261110001822.png)

The sudo command is then used, establishing a reverse shell as the `root` user. The root flag is then retrieved from `/root`.
![](assets/sosimple/Pasted%20image%2020261110002026.png)


## Findings & Remediation

### Social Warfare Remote Code Execution (CVE-2019-9978)
#### Description
The WordPress application uses a vulnerable version of the Social Warfare plugin affected by CVE-2019-9978. The vulnerability allows unauthenticated attackers to supply a malicious remote configuration file through the `swp_url` parameter, which is processed by the plugin and can result in arbitrary PHP code execution.
#### Impact
An unauthenticated attacker can exploit the vulnerability to execute arbitrary code on the web server. In this case, the vulnerability was exploited to establish a reverse shell as the `www-data` user.
#### Remediation
Update the Social Warfare plugin to version 3.5.3 or later, which addresses the vulnerability. Remove the plugin if it is no longer required and ensure that WordPress plugins are regularly updated to versions containing the latest security fixes.

### Insecure SSH Private Key Permissions
#### Description
The private SSH key belonging to the `max` user is stored with permissions that allow all local users to read it.
#### Impact
An attacker who gains access to a local account can retrieve the exposed private key and use it to authenticate through SSH as `max`. In this case, the key was accessed through the compromised `www-data` account and used to establish an SSH session as `max`.
#### Remediation
Restrict access to the private SSH key by setting its permissions to `600` and ensuring that it is owned by the `max` user. Replace the exposed key pair, remove the old public key from authorized access lists, and review other private keys for insecure permissions.

### Sudo Misconfiguration
#### Description
Multiple sudo permissions allow users to execute commands with elevated privileges without providing a password. The `max` user can execute `/usr/bin/service` as `steven`, allowing arbitrary command execution through the service utility. Additionally, the `steven` user can execute `/opt/tools/server-health.sh` as `root`, despite the script being located under a directory writable by `steven`.
#### Impact
An attacker with access to the `max` account can abuse the permitted service command to spawn a shell as `steven`. The second sudo permission can then be exploited to execute commands as `root` when combined with the insecure directory permissions that allow the attacker to create the specified script, resulting in full system compromise.
#### Remediation
Review and restrict sudo permissions to commands that are strictly required by each user. Remove unnecessary permissions and require password authentication for privileged commands where appropriate. Avoid granting sudo access to utilities that permit arbitrary command execution, such as `/usr/bin/service`, and ensure that scripts permitted to run as `root` are stored in directories that cannot be modified by unprivileged users.

### Insecure Directory Permissions
#### Description
The `/opt` directory is owned by the `steven` user and is writable by that account. This allows `steven` to create arbitrary files and subdirectories, including the `/opt/tools/server-health.sh` script referenced in the user's sudo configuration.
#### Impact
An attacker with access to the `steven` account can create a malicious script at `/opt/tools/server-health.sh`. When combined with the existing sudo permission allowing the script to run as `root`, this enables arbitrary command execution with root privileges. In this case, a Bash reverse shell script was created and executed to obtain a root shell.
#### Remediation
Restore ownership of the `/opt` directory to `root` and restrict write permissions to authorized administrators. Ensure that directories containing scripts executed with elevated privileges cannot be modified by unprivileged users. Store privileged scripts in protected locations with appropriate ownership and permissions.