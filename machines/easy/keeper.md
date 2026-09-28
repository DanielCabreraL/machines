# Keeper

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.229.41`

## Summary
**Keeper** is an easy-difficulty Linux machine that highlights default password risks in administrative software, sensitive memory residual leaks in password managers, and key format conversions. Initial access is achieved by discovering default administrative credentials (`root` : `password`) on a Request Tracker (RT) ticketing system, which leaks operational notes containing cleartext SSH credentials for user `lnorgaard`. Privilege escalation to `root` is performed by retrieving a process dump of a password manager, exploiting a KeePass memory dump vulnerability (CVE-2023-32784) to recover the master password, and extracting a PuTTY Private Key (`.ppk`) which is converted to standard OpenSSH PEM format for elevated SSH authentication.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.9p1 (Ubuntu 3ubuntu0.3))
- **80**: HTTP (nginx 1.18.0)

### Web Application Enumeration
Visiting `http://10.129.229.41/` redirects to an internal ticketing interface powered by Best Practical Request Tracker (RT).

1) Attempting default software credentials allows administrative login:

  - Username: `root`
  - Password: `password`

2) Browsing user profiles under Admin -> Users reveals an account for `lnorgaard`.
3) Inspecting the user comments section exposes initial account credentials:

```New user. Initial password set to Welcome2023!```

## Exploitation (Initial Access)

### SSH Authentication
Using the credentials retrieved from the Request Tracker administrative panel, we authenticate via SSH:

```ssh lnorgaard@10.129.229.41```
We gain initial shell access as `lnorgaard` and retrieve `user.txt`.

## Privilege Escalation (Root Flag)

### KeePass Memory Dump Analysis (CVE-2023-32784)
Enumerating the user directory reveals a backup archive `RT30000.zip`. Unzipping the file yields two relevant artifacts:

  - `passcodes.kdbx` — A KeePass 2.x password database.
  - `KeePassDumpFull.dmp` — A process memory dump of the KeePass process.

**Vulnerability Analysis (CVE-2023-32784):**
KeePass 2.x versions prior to 2.54 leave residual plain-text character strings in process memory when master password fields are processed. A memory dump analysis tool (e.g., `keepass-password-dumper`) extracts partial master password character sequences.

Executing a memory dump analysis script against `KeePassDumpFull.dmp` yields partial string matches:

```Possible password pattern: ●rødgrød med fløde```

Reconstructing the complete string via language contextual lookup yields the master password:

**Master Password:** `rødgrød med fløde`

### Database Unlocking & PuTTY Key Conversion
Unlocking `passcodes.kdbx` with the master password exposes entry details for the `root` account. Rather than a standard password, the entry contains a PuTTY Private Key v3 (`.ppk`):

```
PuTTY-User-Key-File-3: ssh-rsa
Encryption: none
Comment: rsa-key-20230519
```

OpenSSH cannot consume .ppk format keys natively without conversion:

1) Save the key content to a local file (`id_rsa.ppk`).
2) Convert the PuTTY key to a standard OpenSSH PEM private key format using `puttygen`:

```puttygen id_rsa.ppk -O private-openssh -o id_rsa```

3) Set appropriate file permissions:

```chmod 600 id_rsa```

4) Authenticate as root via SSH using the converted key:

```ssh -i id_rsa root@10.129.229.41```

Interactive root shell access is established, allowing retrieval of `root.txt`.
