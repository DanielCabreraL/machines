# Netmon

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.71.99`

## Summary
**Netmon** is an easy-difficulty Windows machine that highlights the risks of anonymous FTP access, unsecured configuration backups left on public shares, and administrative vulnerabilities in monitoring software. Initial access is achieved by inspecting residual files on an anonymous FTP server to locate old application configuration files containing rotated administrator credentials for **PRTG Network Monitor**. Logging into the administrative panel enables executing an authenticated Remote Code Execution (RCE) vulnerability within notification parameters to create a new local administrator account. Authenticating via WinRM grants immediate `NT AUTHORITY\SYSTEM` access without requiring further local privilege escalation.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **21**: FTP (Microsoft FTP Service (Anonymous login allowed))
- **80**: HTTP (Indy httpd / PRTG Network Monitor 18.1.37.13946)
- **135**: MSRPC (Microsoft Windows RPC)
- **139**: NetBIOS-SSN (Microsoft Windows NetBIOS-SSN)
- **445**: SMB (Microsoft Windows Server 2008 R2 - 2012 Microsoft-DS)
- **5985**: WinRM (Microsoft HTTPAPI httpd 2.0 (WS-Management))

## FTP Enumeration & Credential Leakage

1) Logging into the FTP server anonymously (`anonymous` : `anonymous`) reveals broad read access to the local drive structure.
2) Navigating through standard user directories allows recovering the initial `user.txt` flag.
3) Exploring system configuration paths reveals an exposed Paessler PRTG configuration directory under `ProgramData`:

  - Path: `\ProgramData\Paessler\PRTG Network Monitor\`

4) Reviewing files within the directory identifies backup configuration files (`PRTG Configuration.old.bak`).
5) Extracting and analyzing the backup configuration file reveals a historical administrator password:

  - **Username:** `prtgadmin`
  - **Legacy Password:** `PrTg@dmin2018`

Testing these credentials directly against the PRTG web application login fails because the password has been updated. Incrementing the year suffix based on standard naming conventions yields the active password:

**Current Password:** `PrTg@dmin2019`

## Exploitation (Initial Access & SYSTEM Access)

### Authenticated PRTG RCE via Notification Scripts
PRTG Network Monitor versions prior to 18.2.38 contain authenticated remote code execution vulnerabilities within the notification handling and ticket script features.

- **Vulnerability Mechanism:** Administrators can configure notification mechanisms that execute local system scripts or commands when alerts trigger. In vulnerable versions, lack of command sanitization in custom notification script parameters allows appending arbitrary command sequences.

- **Execution & Account Creation:**

1) Authenticate to the PRTG web interface using `prtgadmin` : `PrTg@dmin2019`.
2) Create or modify a notification script configuration, injecting system commands into the filename or parameters field to create a new local administrative user:

```test.txt; net user pentest p3nT3st! /add; net localgroup Administrators pentest /add```

3) Trigger the notification or execute the script action to execute the injected commands with system privileges.

### WinRM Authentication
Verifying the user creation via network protocols confirms administrative rights:

```evil-winrm -i 10.129.71.99 -u 'pentest' -p 'p3nT3st!'```

The reverse session establishes successfully, granting direct interactive access under `NT AUTHORITY\SYSTEM` and allowing full access to system administrative files and `root.txt`.
