# Alert

- **IP:** 10.10.11.44
- **OS:** Linux
- **Difficulty:** Easy
- **Date Completed:** 2026-09-02
- **Main Techniques:** `XSS (Markdown)`, `LFI Discovery`, `Hash Cracking`, `Web Service Exploitation (Root)`, `Malicious PHP File (Privesc)`

---

## Summary

Alert is an easy Linux machine that demonstrates a realistic web application attack chain. The initial foothold is achieved by exploiting a stored Cross-Site Scripting (XSS) vulnerability in a Markdown viewer, which is then chained with a Local File Inclusion (LFI) to read sensitive system files. Credentials obtained from an `.htpasswd` file are cracked to gain SSH access. Privilege escalation is accomplished by leveraging group permissions to write a malicious PHP file in a root-executed web service, ultimately granting root access.

---

## Reconnaissance

### Port Scanning

Initial port scan reveals two open ports:

| Port | Service |
|:-----|:--------|
| 22/tcp | SSH |
| 80/tcp | HTTP |

### Web Enumeration

The website is hosted at `alert.htb`. To access it, the IP must be added to `/etc/hosts`: 10.10.11.44 alert.htb


The application is a **Markdown Viewer** with the following sections:
- **Contact Us** - Allows sending messages to the administrator
- **About Us** - General information about the platform
- **Donate** - Donation section

### Subdomain Enumeration

Using `gobuster` to enumerate virtual hosts:

```bash
gobuster vhost -u http://alert.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -t 500
```

A subdomain is discovered: statistics.alert.htb. Navigating to it presents a login prompt, indicating the need for valid credentials.

### Foothold
