# Legacy

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.227.181`

## Summary
**Legacy** is an easy-difficulty Windows machine that illustrates the extreme risks associated with running end-of-life operating systems (Windows XP) and unpatched legacy network services. Initial access and complete administrative control are achieved simultaneously by identifying and exploiting a critical Remote Code Execution vulnerability in the SMBv1 server implementation—specifically **MS17-010** (**EternalBlue** / **CVE-2017-0143**). Because SMB services on legacy Windows installations operate within the system context, successful exploitation grants direct `NT AUTHORITY\SYSTEM` access without requiring local privilege escalation.

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **135**: msrpc (Microsoft Windows RPC)
- **139**: NetBIOS-SSN (Microsoft Windows NetBIOS-SSN)
- **445**: Microsoft-ds (Windows XP Microsoft-DS (SMBv1))

### SMB Service Vulnerability Scanning
Enumerating shares using tools like `smbclient` or `smbmap` fails to list accessible shares due to restrictive permissions. However, running Nmap's SMB vulnerability scripts confirms the presence of unpatched SMBv1 flaws:

```nmap --script "vuln and safe" -p 445 10.129.227.181```

- **Detected Vulnerability:** `MS17-010` (Remote Code Execution vulnerability in Microsoft SMBv1 servers).
- **Target OS:** Windows XP (Windows 5.1).

## Exploitation (Initial Access & SYSTEM Access)

### SMBv1 Remote Code Execution (MS17-010)
MS17-010 addresses multiple remote code execution vulnerabilities in the Microsoft Server Message Block 1.0 (SMBv1) protocol handling.

- **Vulnerability Mechanism:** A buffer overflow vulnerability in the SMBv1 transaction handling logic allows unauthenticated network attackers to write arbitrary kernel memory and execute arbitrary code on the target machine.
- **Named Pipe Checks:** Verifying accessible named pipes identifies the browser named pipe as open and responsive (`32-bit`).
- **Execution & Access:**
  1) A local SMB resource share is hosted using `smbserver.py` to deliver an executable binary or netcat utility (`nc.exe`).
  2) The MS17-010 exploit script targets the accessible `browser` named pipe to invoke a system command (`service_exec`) that fetches `nc.exe` from the attacker's SMB share and executes a reverse shell back to a listening port.
  3) The reverse shell connection is received, granting interactive terminal access.
 
## Privilege Verification
Checking the current privilege context via `whoami` confirms full administrative control:

```nt authority\system```

Because code execution occurred via the SMB kernel driver context, both `user.txt` (located on `C:\Documents` and `Settings\john\Desktop\`) and `root.txt` (located on `C:\Documents and Settings\Administrator\Desktop\`) are directly accessible.
