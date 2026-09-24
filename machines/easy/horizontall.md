# Horizontall

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.59.251`

## Summary
**Horizontall** is an easy-difficulty Linux machine that highlights subdomain enumeration through JavaScript analysis, unauthenticated remote code execution in open-source Content Management Systems (Strapi), and local privilege escalation via a known SUID binary vulnerability (PwnKit). Initial access is achieved by discovering an internal API subdomain (`api-prod.horizontall.htb`), identifying an outdated Strapi CMS installation, and exploiting an unauthenticated password reset and RCE flaw (CVE-2019-18818 / CVE-2019-19609). Privilege escalation to `root` is performed by exploiting a local privilege escalation flaw in the SUID-root binary `pkexec` (PwnKit - CVE-2021-4034).

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating open TCP ports and active service versions:

- **22**: SSH (OpenSSH 7.6p1 (Ubuntu 4ubuntu0.5))
- **80**: HTTP (nginx 1.14.0)

### Subdomain & Web Analysis
Analyzing the main website hosted at `http://horizontall.htb/` reveals static assets. Inspecting the compiled JavaScript files (`/js/app.c68eb462.js`) exposes an unreferenced sub-domain:

```curl -s -X GET http://horizontall.htb/js/app.c68eb462.js | grep -oE '[a-zA-Z0-9.-]+\.htb'```

**Discovered Subdomain:** `http://api-prod.horizontall.htb/`

Fuzzing endpoints on the API subdomain yields several routes:

- `/reviews` — Public reviews endpoint.
- `/admin` — Administration portal interface.
- `/admin/init` — Version endpoint revealing the underlying framework: Strapi 3.0.0-beta.17.4.

## Exploitation (Initial Access)

### Unauthenticated Strapi CMS Remote Code Execution (RCE)
Strapi versions `3.0.0-beta.17.4` and earlier contain critical vulnerabilities in password reset handling and plugin administration.

- **Vulnerability Mechanism:** An attacker can reset the administrator password without prior authentication by sending a crafted password reset request. Once authenticated as admin, the plugin installation endpoint allows executing arbitrary commands or uploading custom plugin code.

- **Execution:**

  1) Triggering the password reset flaw grants administrative session tokens on `http://api-prod.horizontall.htb/`.
  2) Leveraging the admin API endpoint allows executing remote system commands.
  3) Spawning a reverse shell connects back to an active netcat listener, granting initial shell access under the service context (`strapi`).

## Privilege Escalation (Root Flag)

### System & Local Enumeration
Checking internal files in `/home/developer/` yields the `user.txt` flag.

Examining configuration files under /opt/ and application development folders (environments/development/database.json) reveals stored database settings:

**Discovered Account Credentials:** `developer` : `#J!:F9Zt2u`

Checking SUID binaries on the system reveals standard administrative utilities, including `/usr/bin/pkexec`:

```find / -perm -4000 -type f 2>/dev/null```

### Privilege Escalation via PwnKit (CVE-2021-4034)

The system runs an unpatched version of PolicyKit (`polkit`), specifically the `pkexec` executable installed with default SUID-root permissions.

- **Vulnerability Mechanism (PwnKit):** A memory corruption vulnerability in `pkexec` allows unprivileged users to manipulate environment variables (`out-of-bounds` argument processing) when executing commands, forcing pkexec to load and execute an arbitrary shared library as `root`.
- **Privilege Escalation Workflow:**

1) Compile or transfer a standard PwnKit exploit targeting CVE-2021-4034 to a writable directory (`/tmp/`).
2) Executing the exploit binary triggers local privilege escalation without requiring user passwords.

```
cd /tmp
./pwnkit
id
# uid=0(root) gid=0(root) groups=0(root)
```

We obtain an interactive root shell and retrieve `root.txt` from `/root/root.txt`.
