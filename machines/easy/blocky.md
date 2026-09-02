# Blocky

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.10.10.37`

## Summary
**Blocky** is an easy-difficulty Linux machine featuring a WordPress web application showcasing a Minecraft server. Initial access is achieved by discovering exposed custom Java plugins (`.jar files`) inside a web-accessible directory (`/plugins`). Decompiling these plugins reveals hardcoded credentials (`root` / `8YsqfCTnvxAUeduzjNSXe22`) left behind by the server administrator. Privilege escalation is straightforward: the user `notch` shares the same password and possesses full `sudo` privileges, allowing an immediate path to full `root` access.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating open TCP ports and running service detection:

- **21**: FTP (ProFTPD server)
- **22**: SSH (OpenSSH 7.2p2 Ubuntu 4ubuntu2.2)
- **80**: HTTP (Apache httpd 2.4.18)
- **25565**: minecraft (Minecraft Server 1.11.2)

### Web Enumeration & Content Discovery
Inspecting port 80 reveals a WordPress 4.8 site blog post mentioning that the website and Minecraft server are under active development, specifically referencing a "core plugin" being created.

We perform directory brute-forcing using `wfuzz` to uncover hidden endpoints:

```wfuzz -c --hc=404 -t 100 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt http://blocky.htb/FUZZ```

Key Endpoints Identified:

- _/plugins/_ — Directory listing exposed containing Java archives (.jar).
- _/wp-admin/_ — WordPress administration panel login.
- _/phpmyadmin/_ — Database administration interface.
- _/wiki/_ — Empty wiki section.

## Exploitation (User Flag)

### Decompiling Java Plugins & Credential Extraction
Navigating to [http://blocky.htb/plugins/](http://blocky.htb/plugins/) reveals two downloadable Java plugin archives:

- `BlockyCore.jar`
- `griefprevention-1.11.2-3.1.0-216.jar`

We download `BlockyCore.jar` and decompile it using JD-GUI (or `javap` / `cfr`). Inside the `com.el33t.BlockyCore.BlockyCore` class file, we locate hardcoded MySQL database connection parameters:

```
public class BlockyCore {
public String sqlHost = "localhost";
public String sqlUser = "root";
public String sqlPass = "8YsqfCTnvxAUeduzjNSXe22";
}
```

**Extracted Password:** `8YsqfCTnvxAUeduzjNSXe22`

### Initial Access via SSH

Reusing these credentials against system accounts, we authenticate as the user `notch` over SSH:

```ssh notch@10.10.10.37```

Upon successful authentication, we retrieve the `user.txt` flag.

## Privilege Escalation (Root Flag)

### Sudo Privilege Verification

We check the user's groups and `sudo` rights:

```
# uid=1000(notch) gid=1000(notch) groups=1000(notch),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd),110(lpadmin),111(sambashare)

sudo -l
```

The output confirms that `notch` has permission to run `(ALL : ALL) ALL` commands via `sudo`.

### Elevating to Root
We execute `sudo su` using the previously extracted password (`8YsqfCTnvxAUeduzjNSXe22`):

```
sudo su
id
# uid=0(root) gid=0(root) groups=0(root)
```

We obtain full root privileges and capture `root.txt`.

## Conclusions & Key Lessons
- **Hardcoded Credentials:** Secret strings, API keys, and database passwords should never be compiled into public client-side artifacts like Java JAR files or client scripts.
- **Password Reuse:** Reusing administrative database passwords across system user accounts (e.g., SSH / sudo access) drastically increases vulnerability impact.
- **Directory Indexing:** Directory browsing should be disabled on web servers to prevent exposure of sensitive staging files or plugin archives.
