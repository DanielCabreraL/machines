## Mirai

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.69.180`

## Summary
**Mirai** is an easy-difficulty Linux machine (representing a Raspberry Pi IoT environment) that demonstrates the risks of retaining default device credentials and providing overly permissive `sudo` rights. Initial access is achieved by identifying hardware service headers (`X-Pi-hole`) and authenticating via SSH using standard default credentials (`pi` : `raspberry`). Privilege escalation to `root` is immediate due to an unrestricted sudoers policy (`NOPASSWD`: `ALL`). Finally, recovering the missing root flag requires performing raw string carving (`strings`) directly against an unallocated filesystem block device (`/dev/sdb`) to retrieve deleted file data.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 6.7p1 (Debian 5+deb8u3))
- **53**: DNS (dnsmasq 2.76)
- **80**: HTTP (lighttpd 1.4.35)
- **1464**: UPnP (Platinum UPnP 1.0.5.13)
- **32400**: HTTP (Plex Media Server)

### Service & Header Enumeration
1) Attempting to browse the HTTP service on port 80 or inspect the Plex media server interface on port 32400 yields standard landing pages without direct authentication bypass vectors.
2) Inspecting HTTP headers via `curl` reveals identifying platform architecture:

```curl -I http://10.129.69.180```

**Key Header:** X-Pi-hole: A black hole for Internet advertisements.

Hardware documentation and standard support forums identify the default factory credentials for this embedded distribution as `pi` : `raspberry`.

## Exploitation (Initial Access)

### Default Credential SSH Access
Using the discovered default hardware credentials, we authenticate directly via SSH:

```ssh pi@10.129.69.180```

**Credentials:** `pi` : `raspberry`

We obtain initial user shell access and can retrieve `user.txt` from the user home directory.

## Privilege Escalation & Forensic Flag Recovery

### Sudo Privilege Enumeration
Checking assigned `sudo` rights for the current user via `sudo -l` reveals an insecure administrative configuration:

```
User pi may run the following commands on localhost:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

Switching to the root user context requires no password authentication:

```sudo su```

### Raw Storage Carving for Deleted Flags
Upon entering the `/root/` directory, the standard `root.txt` file is missing. Instead, an administrative note is present:

```"I lost my original root.txt! I think I may have a backup on my USB stick..."```

Navigating to the attached USB mount point (`/media/usbstick`) reveals a note indicating the files were accidentally deleted.

**Recovery Mechanism:** Deleting files from standard filesystems (`ext4`) removes file allocation pointers from the journal, but the underlying data blocks remain on the storage medium until overwritten. Raw string carving using the `strings` utility directly against the block device reveals the residual file content:

```strings /dev/sdb```

Scanning the output locates the plaintext contents of the missing `root.txt` flag.
