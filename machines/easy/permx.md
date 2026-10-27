# PermX

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.76.143`

## Summary
**Permx** is an easy-difficulty Linux machine that illustrates risks associated with unpatched e-learning platforms, credential reuse across system accounts, and misconfigured `setfacl` administrative scripts. Initial access is achieved by enumerating a virtual subdomain (`lms.permx.htb`), identifying an outdated instance of **Chamilo LMS (v1.11.24)**, and exploiting a public Remote Code Execution (RCE) vulnerability (**CVE-2023-4226**). Lateral movement to user mtz is accomplished by recovering database credentials from configuration files and leveraging password reuse. Privilege escalation to `root` is completed by exploiting an administrative script (`acl.sh`) that allows appending permissions or linking system files (such as `/etc/passwd`) via Access Control Lists.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.9p1 (Ubuntu 3ubuntu0.10))
- **80**: HTTP (Apache httpd 2.4.52)

### Subdomain & Web Enumeration
1) Inspecting the main web application on port 80 reveals a corporate eLearning landing page and contact details referencing `permx@htb.com`.
2) Fuzzing for virtual hosts yields an active LMS subdomain:

**Discovered Subdomain:** `lms.permx.htb`

3) Further directory fuzzing on `lms.permx.htb` locates documentation and version assets (`/documentation/`), confirming the application runs **Chamilo LMS 1.11**.

## Exploitation (Initial Access)

### Chamilo LMS Remote Code Execution (CVE-2023-4226)
Chamilo LMS versions up to 1.11.24 contain an unauthenticated file upload / RCE vulnerability in the `/main/upload/` and public assessment endpoints.

- **Vulnerability Mechanism:** Improper sanitization in upload and plugin mechanisms allows unauthenticated remote attackers to write arbitrary executable PHP files into web-accessible directories.

- **Execution:**

1) Utilizing a public exploit script targeting CVE-2023-4226 against `http://lms.permx.htb/`.
2) The exploit uploads a PHP reverse shell payload and triggers execution, establishing an active shell under the web service user context (`www-data`).

## Lateral Movement (mtz)

### Database Configuration & Credential Reuse
1) Locating configuration files within the Chamilo application directory structure:

```
find . -name \*config\* | grep -vE "Resources|symfony|vendor|MathJax|plugin"
```

2) Inspecting ./app/config/configuration.php reveals database connection strings and credentials:

**Database User:** `chamilo`
**Password:** `03F6lY3uXAP2bkW8`

3) Authenticating to the local MySQL instance (`mysql -uchamilo -p`) permits querying user password hashes from the database. While cracking the administrator hash via `hashcat` proves unnecessary, testing the database password against system users (`mtz`) succeeds.

4) Logging in as user `mtz` via SSH (or switching user context) grants access to user files and the `user.txt` flag.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Checking assigned `sudo` execution rights for user `mtz` via `sudo -l` reveals access to a custom administrative script:

```
User mtz may run the following commands on permx:
    (ALL : ALL) NOPASSWD: /opt/acl.sh
```

### Analyzing /opt/acl.sh
Inspecting the script logic reveals how file permissions are modified:

- The script accepts three parameters: user, permissions, and target file.
- It validates that the target file resides within `/home/mtz/` and does not contain directory traversal sequences (`..`).
- It checks that the target is a regular file (`-f`) and executes `sudo setfacl` to apply the requested permissions.

### ACL Symlink Exploitation (/etc/passwd)
Because the script validates files using `-f` (which follows symbolic links), a symbolic link pointing to a restricted system file can be placed inside the user's home directory.

- **Privilege Escalation Workflow:**
  1) Create a symbolic link in the home directory pointing to `/etc/passwd`:

```
ln -s /etc/passwd /home/mtz/passwd
```

  2) Use the sudo-restricted script to grant read/write access (`rwx`) over the linked file:

```
sudo /opt/acl.sh mtz rwx /home/mtz/passwd
```

  3) Because the ACL grants write access to `/etc/passwd`, append a new user entry with root privileges (UID 0) or update the root password hash directly:

```
openssl passwd -1 -salt hack password123
# Append new root-equivalent user or modify root line:
echo 'hacker:$1$hack$1vK...:0:0::/root:/bin/bash' >> /etc/passwd
```

  4) Switch to the newly created account or use the modified root credentials to spawn an administrative shell.

Full administrative root access is acquired, allowing retrieval of `root.txt` from `/root/root.txt`.
