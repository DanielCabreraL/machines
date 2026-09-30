# Lame

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.63.54`

## Summary
**Lame** is a classic easy-difficulty Linux machine on Hack The Box that demonstrates the risks of running outdated, unpatched SMB services with default configuration flaws. While initial port enumeration reveals several legacy services (including FTP and SSH), initial access is achieved directly by exploiting an unauthenticated remote command execution vulnerability (**CVE-2007-2447**) in Samba 3.0.20 via the username map `script configuration parameter`. Because the SMB daemon runs with administrative privileges, successful exploitation immediately provides `root` access without requiring local privilege escalation.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **21**: FTP (vsftpd 2.3.4 (Anonymous login allowed))
- **22**: SSG (OpenSSH 4.7p1 (Debian 8ubuntu1))
- **139**: NetBIOS-SSN (Samba smbd 3.X - 4.X)
- **445**: SMB (Samba smbd 3.0.20-Debian)
- **3632**: Distributed Compiler (distccd v1)

### Service Enumeration

1) **FTP (Port 21):** Anonymous authentication is permitted (`anonymous` : `anonymous`), but directory listing yields no actionable files or usable data.
2) **Samba (Ports 139/445):** Version enumeration identifies Samba 3.0.20-Debian.

## Exploitation (Initial Access & Root Flag)

### Samba username map script Command Execution (CVE-2007-2447)
Samba versions `3.0.20` through `3.0.25rc3` contain a remote command execution vulnerability when the `username map script` option is enabled in `smb.conf`.

- **Vulnerability Mechanism**: When authenticating over SMB, if a username containing shell metacharacters (e.g., backticks or `$()`) is submitted, Samba fails to properly sanitize the input before passing it to the configured `username map script` shell invocation. This allows unauthenticated attackers to execute arbitrary system commands.
- **Execution & Privilege Access:**
  1) An SMB connection is initiated, specifying a crafted username payload containing a shell command construct (e.g., `/=nohup`).
  2) The embedded command triggers an outbound reverse connection back to an active network listener.
  3) Because `smbd` executes under the high-privilege `root` context, the resulting interactive shell immediately inherits full administrative privileges (`uid=0(root)`).

  ```
  id
  uid=0(root) gid=0(root) groups=0(root)
  ```
Both `user.txt` (located in `/home/`) and `root.txt` (located in `/root/`) are directly accessible.
