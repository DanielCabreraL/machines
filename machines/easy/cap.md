# Cap

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.10.10.245`

## Summary
**Cap** is an easy-difficulty Linux machine featuring a web dashboard that provides network diagnostics and packet captures. Initial access is achieved by identifying an **Insecure Direct Object Reference (IDOR)** vulnerability on the PCAP download endpoint (`/data/0`), allowing us to download a packet capture containing unencrypted FTP credentials for the user `nathan`. Privilege escalation is accomplished by exploiting excessive Linux Capabilities assigned to the Python 3.8 binary (`cap_setuid`), allowing arbitrary UID manipulation to execute commands as `root`.

## Reconnaissance

### Port Scanning (Nmap)
We begin by scanning open TCP ports and identifying active service versions:

- **21**: FTP (vsftpd 3.0.3)
- **22**: SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.2)
- **80**: HTTP (Gunicorn (Python Web Server)

### Web Enumeration & IDOR Identification
Navigating to the web interface on port 80 reveals several functional endpoints:

- _/ip_ — Displays output of network interface configuration (`ifconfig`).
- _/netstat_ — Displays active network connections.
- _/capture_ — Triggers a packet capture sequence and redirects to a results page (`/data/1`).

Inspecting the URL structure `/data/{id}` suggests a potential **IDOR (Insecure Direct Object Reference)** flaw in transaction indexing.

## Exploitation (User Flag)

### Exploiting IDOR & PCAP Analysis

- Decrementing the path ID from `/data/1` to `/data/0` yields a previously recorded network traffic capture file containing administrative traffic.
- Downloading 0.pcap and inspecting its contents via Wireshark (or `tshark` / `strings`) reveals unencrypted credentials transmitted during an FTP authentication attempt:

```
USER nathan
PASS Buck3tH4TF0RM3!
```

**Extracted Credentials:** `nathan` : `Buck3tH4TF0RM3!`

### Initial Access via SSH
We authenticate using the recovered credentials over SSH for a stable interactive terminal:

```ssh nathan@10.10.10.245```

Upon logging in, we retrieve the `user.txt` flag from `/home/nathan/user.txt`.

## Privilege Escalation (Root Flag)

### Linux Capability Enumeration

We enumerate binary capabilities across the filesystem using `getcap`:

```getcap -r / 2>/dev/null```

**Key Finding:**

```/usr/bin/python3.8 = cap_net_bind_service,cap_setuid+eip```

The `/usr/bin/python3.8` binary has been assigned the `cap_setuid` capability with effective, permitted, and inheritable flags (`+eip`). This allows the binary to set arbitrary user IDs (including UID 0 for `root`).

### Abusing Linux Capabilities for Root Access

We leverage the `setuid` function within Python 3.8 to elevate our process privileges to root:

```python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'```

Checking process ownership confirms effective `root` privileges:

```
id
# uid=0(root) gid=1000(nathan) groups=1000(nathan)
```

We obtain full `root` access and capture `root.txt`.

## Conclusions & Key Lessons
- **Insecure Direct Object References (IDOR):** Direct object references must enforce strict server-side access control validation to prevent unauthorized users from accessing sensitive transaction data or captures.
- **Cleartext Protocols:** Sensitive credentials transmitted over unencrypted protocols (such as standard FTP) are vulnerable to interception and extraction via packet inspection.
- **Principle of Least Privilege with Capabilities:** Administrative POSIX capabilities (such as cap_setuid) should never be assigned to scripting interpreters or general-purpose execution environments like Python.
