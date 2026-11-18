# SPX (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan identifies the host as a Linux machine with SSH running on port 22 and an Apache web server running on port 80.
![](assets/spx/Pasted%20image%2020261115172407.png)

The web application features a Tiny File Manager application. The default credentials for the admin user are `admin:admin123`, but these credentials are not used in this case.
![](assets/spx/Pasted%20image%2020261115172456.png)

The `phpinfo.php` file is accessible.
![](assets/spx/Pasted%20image%2020261115172645.png)

An SPX key is found at the bottom of the `phpinfo` page.
![](assets/spx/Pasted%20image%2020261115172704.png)

A Google search leads to a [GitHub issue page](https://github.com/NoiseByNorthwest/php-spx/issues/251) revealing a path traversal vulnerability in SPX and a proof of concept.
![](assets/spx/Pasted%20image%2020261115172755.png)

The request is modified to include the SPX key found in `phpinfo`. The web application returns the content of `/etc/passwd`, revealing a local `profiler` user.
![](assets/spx/Pasted%20image%2020261115173549.png)
## Initial Access
The Tiny File Manager application is run entirely through the `index.php` file, which also contains users' credentials. The SPX path traversal vulnerability is abused to read the source code of `index.php`, revealing credentials for `admin` and `user`.
![](assets/spx/Pasted%20image%2020261115173022.png)

The `admin` user's password hash is cracked using John the Ripper. The password is used to access the web application's dashboard.
![](assets/spx/Pasted%20image%2020261115173211.png)

The PHP reverse shell from PentestMonkey is modified to point to port 443 on the attacking machine.
![](assets/spx/Pasted%20image%2020261115173321.png)

The reverse shell is then uploaded through Tiny File Manager.
![](assets/spx/Pasted%20image%2020261115173359.png)

The file is then accessed directly through the browser at `http://192.168.150.108/php-reverse-shell.php`. The web application executes the script, establishing a reverse shell as `www-data`.
![](assets/spx/Pasted%20image%2020261115173437.png)
## Privilege Escalation
The password of the web application's `admin` user is reused to switch to the previously identified `profiler` account.
![](assets/spx/Pasted%20image%2020261115173627.png)

The `sudo -l` command reveals that the `profiler` account can execute the following command through sudo.
```
/usr/bin/make install -C /home/profiler/php-spx
```
![](assets/spx/Pasted%20image%2020261115173813.png)

The `Makefile` in the specified directory is writable by the compromised user.
![](assets/spx/Pasted%20image%2020261115173839.png)

The file is replaced with a malicious one, using the `install` target to execute `chmod u+s /bin/bash`. This sets the SUID bit on the binary, allowing any user to execute it with the `-p` flag to maintain privileges.
![](assets/spx/Pasted%20image%2020261115173942.png)

The sudo command is then executed. The output shows the execution of the `chmod` command.
![](assets/spx/Pasted%20image%2020261115174030.png)

Inspection of `/bin/bash` shows that the binary has the SUID bit set.
![](assets/spx/Pasted%20image%2020261115174109.png)

A root shell is obtained by executing `/bin/bash -p`.
![](assets/spx/Pasted%20image%2020261115174131.png)
## Finding & Remediation
### Sensitive Information Disclosure
#### Description
The `phpinfo.php` file is publicly accessible and exposes sensitive PHP configuration information, including the SPX authentication key.
#### Impact
An unauthenticated attacker can retrieve the SPX authentication key and use it to access restricted SPX functionality. In this case, the exposed key enabled exploitation of the SPX path traversal vulnerability.
#### Remediation
Remove publicly accessible `phpinfo.php` files from production environments. Restrict access to debugging and profiling interfaces to authorized users and rotate the exposed SPX authentication key.
### SPX Path Traversal (CVE-2024-42007)
#### Description
The PHP-SPX extension is vulnerable to CVE-2024-42007, a path traversal vulnerability caused by insufficient validation of user-controlled file paths. The vulnerability allows files outside the intended directory to be accessed through the SPX web interface.
#### Impact
An attacker with access to the SPX interface can read sensitive files accessible to the web server process. In this case, exploitation of the vulnerability exposed the contents of `/etc/passwd` and the Tiny File Manager source code, including user password hashes.
#### Remediation
Upgrade PHP-SPX to version 0.4.16 or later, which addresses CVE-2024-42007. Restrict access to the SPX web interface and disable profiling functionality in production environments unless explicitly required.
### Password Reuse
#### Description
The Tiny File Manager administrative account and the local `profiler` account share the same password, despite belonging to separate authentication mechanisms.
#### Impact
An attacker who obtains the web application's administrative password can reuse it to access the local `profiler` account. In this case, password reuse enabled access to the `profiler` account from the compromised `www-data` account.
#### Remediation
Enforce unique passwords across application and operating system accounts. Rotate the reused password on both accounts and ensure that administrative credentials are not shared between different services.
### Sudo Misconfiguration
#### Description
The `profiler` account is permitted to execute `/usr/bin/make install -C /home/profiler/php-spx` with root privileges through sudo. The `Makefile` in the specified directory is writable by the `profiler` user, allowing the commands associated with the `install` target to be modified.
#### Impact
An attacker with access to the `profiler` account can modify the `Makefile` to execute arbitrary commands with root privileges. In this case, exploitation resulted in a SUID-enabled Bash binary and a root shell.
#### Remediation
Remove the unnecessary sudo permission for `make`. If privileged installation operations are required, ensure that the Makefile, associated build files and their parent directories are owned and controlled by trusted administrative users. Avoid executing build or installation commands with elevated privileges from directories controlled by unprivileged users.
