# Love

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.48.103`

## Summary
**Love** is an easy-difficulty Windows machine that highlights the risks of Server-Side Request Forgery (SSRF), insecure internal service exposure, and privilege escalation via Windows installer policies. Initial access is achieved by identifying an SSRF vulnerability on a sub-domain (`staging.love.htb`), which allows querying an internal administrative interface running on port 5000 to retrieve cleartext credentials. Using these credentials to log into the main Voting System application, authenticated file upload capabilities are exploited to execute arbitrary PHP code. Local privilege escalation to `NT AUTHORITY\SYSTEM` is completed by leveraging the AlwaysInstallElevated registry policy to execute a crafted Windows Installer (`.msi`) package with administrative privileges.

## Reconnaissance

### Port Scanning (Nmap)
Enumeration reveals multiple open services on the host:

- **80**: HTTP (Apache httpd 2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27)
- **135**: MSRPC (Microsoft Windows RPC)
- **139**: NetBIOS-SSN (Microsoft Windows NetBIOS-SSN)
- **443**: HTTPS (Apache httpd 2.4.46 (Win64))
- **445**: SMB (Windows 10 Pro 19042 Microsoft-DS)
- **3306**: MySQL (MariaDB 10.3.24)
- **5000**: HTTP (Apache httpd 2.4.46 (Internal Admin / Restricted Access))

### Web Directory & Subdomain Enumeration
1) Fuzzing the web root on port 80 reveals administrative endpoints associated with a Voting System application (`/admin/`, `/login.php`).
2) Virtual host enumeration / SSL certificate inspection identifies a secondary host: `staging.love.htb`.

## Exploitation (Initial Access)

### SSRF to Internal Port 5000
The web application hosted on `staging.love.htb` includes a URL scanner feature intended to fetch and display external file contents.

- **Vulnerability Mechanism:** The application fails to validate or restrict destination IP addresses provided to the URL scanner, creating a Server-Side Request Forgery (SSRF) flaw.
- **Internal Reconnaissance:** Submitting `http://127.0.0.1:5000` forces the server to make a local HTTP request to its own loopback interface on port 5000, bypassing external access controls.
- **Credential Disclosure:** The HTTP response returned from internal port 5000 displays cleartext administrator credentials:

```admin : @LoveIsInTheAir!!!!```

### Authenticated Remote Code Execution (RCE)
1) Authenticating to the Voting System admin panel ([`http://10.129.48.103/admin/`) using the recovered credentials grants access to the management dashboard.
2) The administration interface allows uploading profile images or application assets.
3) Uploading a PHP script disguised or processed through the image upload functionality allows arbitrary code execution upon navigating to the uploaded file's path in `/images/`.
4) Spawning a reverse connection grants interactive shell access as user `phoebe`.

## Privilege Escalation (Root Flag)

### AlwaysInstallElevated Enumeration
Enumerating local security configurations using system auditing scripts (`winPEAS`) reveals that the AlwaysInstallElevated policy is enabled in the Windows Registry across both user and system scopes:

```
HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
```

**Vulnerability Mechanism:** When both registry values are set to `1`, the Windows Installer service (`msiexec.exe`) executes `.msi` packages with elevated `NT AUTHORITY\SYSTEM` privileges, regardless of the privileges of the user launching the installation.

### MSI Package Execution
1) A Windows Installer (`.msi`) package designed to execute an elevated command or shell payload is generated.
2) The file is transferred to the target host (e.g., using `certutil.exe`).
3) Executing the installer silently via `msiexec` spawns a high-privilege shell:

```msiexec /quiet /qn /i reverse.msi```

4) An incoming connection is received, establishing interactive terminal access as `NT AUTHORITY\SYSTEM` and allowing full access to system flags (`user.txt` and `root.txt`).
