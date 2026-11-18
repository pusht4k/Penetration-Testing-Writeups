# Lavita (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results show port 22 open with SSH and port 80 open with an Apache web server running.
![](assets/lavita/Pasted%20image%2020261110164150.png)

The web application's home page displays a static image with a few menu options.
![](assets/lavita/Pasted%20image%2020261110164333.png)

A login page is accessible through the `Dropdown` menu under `Demo`. Attempts at guessing credentials fail.
![](assets/lavita/Pasted%20image%2020261110164422.png)

A `gobuster` scan reveals a registration endpoint at `/register`.
![](assets/lavita/Pasted%20image%2020261110164624.png)


After successful registration and authentication, an Image Upload feature is found in the dashboard.
![](assets/lavita/Pasted%20image%2020261110164710.png)

A picture is uploaded and an attempt to access it fails. The 404 page reveals that the application is Laravel version 8.4.0.
![](assets/lavita/Pasted%20image%2020261110164802.png)
## Initial Access
This application is vulnerable to a remote code execution exploit through DEBUG mode (CVE-2021-3129). A publicly available proof of concept script is found on [joshuavanderpoll's GitHub](https://github.com/joshuavanderpoll/CVE-2021-3129).
![](assets/lavita/Pasted%20image%2020261110164902.png)

The script is used to execute the `whoami` command on the target. The output reveals that commands are executed under the `www-data` user.
![](assets/lavita/Pasted%20image%2020261110165134.png)

A Bash reverse shell one-liner is encoded in Base64.
![](assets/lavita/Pasted%20image%2020261110165212.png)

The script is then used once again to execute the reverse shell through use of `echo` to pass the string to `base64 -d` through a pipe, before using a second pipe to pass it to `bash`.
![](assets/lavita/Pasted%20image%2020261110165257.png)

This establishes a reverse shell as `www-data`.
![](assets/lavita/Pasted%20image%2020261110165359.png)
## Privilege Escalation
The `pspy32` tool is used, exposing a PHP process that executes `/var/www/html/lavita/artisan` regularly under UID 1001.
![](assets/lavita/Pasted%20image%2020261110165538.png)

Since the `/var/www/html` directory is owned by `www-data`, the file could be replaced with a malicious PHP script instead.
![](assets/lavita/Pasted%20image%2020261110165652.png)

The `php-reverse-shell.php` script from PentestMonkey is configured to point to port 443 on the attacking machine.
![](assets/lavita/Pasted%20image%2020261110165738.png)

The script is transferred to the target system, replacing the original `artisan` file, which has been moved to a backup. The new file is then given execution permissions.
![](assets/lavita/Pasted%20image%2020261110165904.png)

Shortly after, a reverse shell is established as the user `skunk`. The user is shown as a member of the `27(sudo)` group.
![](assets/lavita/Pasted%20image%2020261110165936.png)

Python's `pty` library is used to spawn a TTY shell. The `sudo -l` command is then used to enumerate sudo permissions, revealing that `skunk` can either run any command using `sudo` with a password or execute the `/usr/bin/composer --working-dir=/var/www/html/lavita *` command without one.
![](assets/lavita/Pasted%20image%2020261110170107.png)

GTFOBins documents a privilege escalation vector using `composer` by creating a malicious `composer.json` file that executes `/bin/sh`.
![](assets/lavita/Pasted%20image%2020261110170413.png)

The `composer.json` file in the `/var/www/html/lavita` directory is owned by the `www-data` user.
![](assets/lavita/Pasted%20image%2020261110170446.png)

The previously established `www-data` reverse shell is used to overwrite the file with the malicious script.
![](assets/lavita/Pasted%20image%2020261110170534.png)

Since the sudo command uses a wildcard, the `run-script x` option is added to the command to run the malicious script and spawn a `root` shell.
![](assets/lavita/Pasted%20image%2020261110170636.png)

## Finding & Remediation
### CVE-2021-3129 - Laravel Remote Code Execution
#### Description
The web application runs Laravel version 8.4.0 with debug mode enabled, exposing it to CVE-2021-3129, an unauthenticated remote code execution vulnerability.
#### Impact
An unauthenticated attacker can exploit this vulnerability to execute arbitrary commands on the target system, establishing a reverse shell as the `www-data` user.
#### Remediation
Upgrade Laravel to version 8.4.3 or later and the Ignition package to version 2.5.2 or later. Disable debug mode in production environments and ensure that sensitive debugging functionality is not publicly accessible.
### Misconfigured Automated Process
#### Description
An automated process periodically executes `/usr/bin/php /var/www/html/lavita/artisan` as the `skunk` user. However, the `artisan` file is located in a directory controlled by `www-data`, allowing the file to be replaced with a malicious PHP script.
#### Impact
An attacker with access to the `www-data` account can replace the `artisan` file with a malicious script, which is automatically executed with the privileges of `skunk`. This allows the attacker to execute arbitrary commands and establish a reverse shell as `skunk`.
#### Remediation
Restrict write permissions on the `artisan` file and its parent directories to trusted administrative accounts. Ensure that automated processes do not execute files that can be modified by lower-privileged users.
### Sudo Misconfiguration
#### Description
The `skunk` user is permitted to execute `/usr/bin/composer --working-dir=/var/www/html/lavita *` using `sudo` without providing a password. The specified directory contains a `composer.json` file controlled by `www-data`, while the wildcard allows additional arguments to be supplied to the command.
Additionally, `skunk` is permitted to execute any command using `sudo` when providing a password.
#### Impact
An attacker with access to the `www-data` and `skunk` accounts can modify `composer.json` to include a malicious script and execute it through the permitted `sudo` command. This allows arbitrary command execution with `root` privileges, resulting in complete system compromise.
#### Remediation
Remove the `sudo` permission for Composer and avoid using wildcards in privileged command configurations. Restrict write access to `composer.json` and other files used by privileged processes. Review whether `skunk` requires unrestricted `(ALL:ALL)` sudo permissions and apply the principle of least privilege.