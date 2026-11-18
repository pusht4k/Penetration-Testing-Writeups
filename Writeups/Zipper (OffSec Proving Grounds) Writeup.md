# Zipper (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
The Nmap scan results show only two ports open on the target, with SSH running on port 22 and an Apache web server running on port 80.
![](assets/zipper/Pasted%20image%2020261115194455.png)

The web application contains a file upload feature. Uploaded files are zipped and a download link is provided.
![](assets/zipper/Pasted%20image%2020261115194716.png)

The web application is using the `file` parameter to display pages.
![](assets/zipper/Pasted%20image%2020261115194832.png)

The target is vulnerable to local file inclusion. Using PHP wrappers, the source code of `index.php` is obtained in Base64.
![](assets/zipper/Pasted%20image%2020261115194933.png)

CyberChef is used to decode the string, revealing that the web application uses the `include()` PHP function to display pages after adding the `.php` extension.
![](assets/zipper/Pasted%20image%2020261115195027.png)

A subsequent query is sent to obtain the source code of `upload.php`.
![](assets/zipper/Pasted%20image%2020261115195118.png)

The decoded code reveals that uploaded files are saved to the `/uploads` directory with the `upload_` prefix, followed by the current time and the `.zip` extension.
![](assets/zipper/Pasted%20image%2020261115195146.png)
## Initial Access
A PHP reverse shell script is stored in an archive and uploaded to the web application. When hovering over the `Click here to download the zip file` button the filename and path are shown in the bottom left of the browser.
![](assets/zipper/Pasted%20image%2020261115195337.png)

The file is then directly executed from within the stored archive using the `phar://` PHP wrapper.
![](assets/zipper/Pasted%20image%2020261115195424.png)

A reverse shell is established as `www-data`.
![](assets/zipper/Pasted%20image%2020261115195504.png)
## Privilege Escalation
Inspection of `/etc/crontab` reveals a cron job executing `/opt/backup.sh` as root.
![](assets/zipper/Pasted%20image%2020261115195610.png)

The script uses a password variable that is set to the content of `/root/secret`. The script then uses `cd` to move to the `uploads` directory and delete all temporary files before running `7za` to create an archive using the password and redirect the output to a log file.
```
$ cat /opt/backup.sh 
#!/bin/bash
password=`cat /root/secret`
cd /var/www/html/uploads
rm *.tmp
7za a /opt/backups/backup.zip -p$password -tzip *.zip > /opt/backups/backup.log
```

To exploit this, a file can be created with a symbolic link to `/root/secret`. A second file can then be created with the `@` prefix before using the same name. The `@` symbol is used by `7za` to read files, this should result in the content of `/root/secret` being added to the log file as a filename.
The `pusht4k.zip` and `@pusht4k.zip` files are created in the uploads directory.
![](assets/zipper/Pasted%20image%2020261115200218.png)

Shortly after, the `backup.log` file is read, revealing an odd string that seems to be a password.
![](assets/zipper/Pasted%20image%2020261115200258.png)

The string is used with `su` to switch to the root user.
![](assets/zipper/Pasted%20image%2020261115200400.png)

## Finding & Remediation
### Local File Inclusion
#### Description
The web application uses the `include()` PHP function to load pages based on the user-controlled `file` parameter without sufficient input validation. The application also permits the use of PHP wrappers within the parameter.
#### Impact
An attacker can access sensitive local files and application source code. In this case, the vulnerability exposed the application's file upload implementation and allowed execution of uploaded PHP code, resulting in a reverse shell as `www-data`.
#### Remediation
Replace dynamic file inclusion with a predefined allowlist of permitted pages. Prevent user-controlled input from being passed directly to the `include()` function. Store uploaded files outside the web root and prevent them from being included or executed by the application.
### Wildcard Argument Injection
#### Description
A cron job running with root privileges executes `/opt/backup.sh`, which uses a wildcard to include files from a directory writable by unprivileged users. The backup command does not distinguish between filenames and special command-line arguments interpreted by `7za`.
#### Impact
An attacker with write access to the uploads directory can manipulate the privileged backup process and cause sensitive file contents to be disclosed through its log file. In this case, exploitation exposed the root password, resulting in full root access to the target.
#### Remediation
Avoid using unrestricted wildcard expansion when processing files from directories writable by unprivileged users. Validate filenames, reject symbolic links and ensure that filenames cannot be interpreted as command-line arguments. Restrict access to backup logs and execute the backup process with the minimum privileges required. Use a dedicated backup password instead of reusing the root account's password.
