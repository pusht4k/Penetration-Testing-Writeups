# Vegeta1 (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
An aggressive Nmap scan reveals two open ports on the target. SSH is running on port 22 while an Apache web server is running on port 80.
![](assets/vegeta/Pasted%20image%2020261109233715.png)

The web application's home page displays a static image of the prince of all Saiyans.
![](assets/vegeta/Pasted%20image%2020261109233820.png)

The `/find_me` entry is found in the `robots.txt` file.
![](assets/vegeta/Pasted%20image%2020261109233919.png)

While the page seems empty, a Base64-encoded string is found at the bottom of the page source.
![](assets/vegeta/Pasted%20image%2020261109234012.png)

CyberChef is used to decode the string twice. The Render Image option is then used to reveal a QR code.
![](assets/vegeta/Pasted%20image%2020261109234110.png)

Using `zbarimg` on the QR code reveals the password: `topshellv`. However, this password is tested through SSH and doesn't lead to successful authentication.
![](assets/vegeta/Pasted%20image%2020261109234158.png)
## Initial Access
Based on the Dragon Ball theme of the website, the `/bulma` directory is manually guessed, revealing an enabled directory listing containing a `hahahaha.wav` file.
![](assets/vegeta/Pasted%20image%2020261109234303.png)

The file includes audio that sounds like Morse code. An online Morse code translator is used, revealing a `trunks` user with the password `U$3R`.
![](assets/vegeta/Pasted%20image%2020261109234502.png)

These credentials are used to successfully authenticate to the target through SSH as `trunks`.
![](assets/vegeta/Pasted%20image%2020261109234709.png)
## Privilege Escalation
Enumeration of `/etc/passwd` reveals that `trunks` is the owner of the file and has write permissions over it.
![](assets/vegeta/Pasted%20image%2020261109234806.png)

In addition, the `.bash_history` file is found in `trunks`' home directory, showing a Perl command to hash the password `Password@973`, followed by a subsequent `echo` command to append the `Tom` user to the `/etc/passwd` file with `UID 0`, which translates to root.
![](assets/vegeta/Pasted%20image%2020261109235035.png)

The command is reused to add the fake root user to `/etc/passwd`.
![](assets/vegeta/Pasted%20image%2020261109235056.png)

The `su` command is then used to switch to the `Tom` user, obtaining root-level access to the system.
![](assets/vegeta/Pasted%20image%2020261109235125.png)

## Findings & Remediation

### Sensitive Information Disclosure
#### Description
The web server exposes a `hahahaha.wav` file through the `/bulma` directory, which has directory listing enabled. The audio file contains Morse code that reveals valid SSH credentials for the `trunks` user.
#### Impact
An unauthenticated attacker can access the exposed file, decode the Morse code, and recover the credentials for the `trunks` account. These credentials can then be used to gain access to the underlying system through SSH.
#### Remediation
Remove sensitive information, including credentials, from publicly accessible files and directories. Disable directory listing and restrict access to files that are not intended to be publicly available. Rotate the exposed password and ensure that credentials are stored securely rather than embedded in application resources.
### Insecure File Permissions
#### Description
The `/etc/passwd` file is owned by the `trunks` user and has write permissions assigned to that account. This allows a non-privileged user to modify the system's account configuration, including adding new users with `UID 0`.
#### Impact
An attacker with access to the `trunks` account can modify `/etc/passwd` to create an additional account with `UID 0`, granting it root-level privileges. In this case, the tester added the `Tom` account and used `su` to obtain root-level access to the system.
#### Remediation
Restore the ownership of `/etc/passwd` to `root` and restrict write permissions to the root account only. Review other sensitive system files for improper ownership or permissions.