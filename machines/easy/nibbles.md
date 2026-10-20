# Nibbles

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.73.120`

## Summary
**Nibbles** is an easy-difficulty Linux machine that illustrates the risks of directory information disclosure, weak default administrative passwords, and insecure `sudo` permissions linked to non-existent script paths. Initial access is achieved by identifying a hidden blog application directory (`/nibbleblog/`), discovering exposed configuration files containing weak default credentials (`admin` : `nibbles`), and leveraging an authenticated file upload flaw in the My Image plugin to execute remote code. Privilege escalation to `root` is performed by exploiting a `sudo` rule referencing a monitoring script inside a user-writable path that does not yet exist, allowing a custom malicious script to be placed and executed with root privileges.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 7.2p2 (Ubuntu 4ubuntu2.2))
- **80**: HTTP (Apache httpd 2.4.18)

### Web Enumeration & Information Disclosure
1) Visiting the main web page hosted on port 80 displays a basic "Hello World!" message. Inspecting the HTML source code reveals a hidden comment:
`<!-- /nibbleblog/ directory. Nothing interesting here! -->`

2) Navigating to `http://10.129.73.120/nibbleblog/` exposes the actual blogging application.
3) Fuzzing the application paths uncovers administrative endpoints and sensitive configuration directories:

- `/nibbleblog/admin.php` — Administrator login portal.
- `/nibbleblog/content/private/config.xml` — System configuration file.
- `/nibbleblog/content/private/users.xml` — User profile data.

4) Reviewing the contents of `users.xml` and configuration files confirms the presence of the administrative user (`admin`). Guessing standard default credentials (matching the machine name: `admin` : `nibbles`) successfully authenticates to the admin panel.

## Exploitation (Initial Access)

### Nibbleblog 4.0.3 Arbitrary File Upload
Nibbleblog versions prior to 4.0.3 contain an arbitrary file upload vulnerability via the My Image plugin.

- **Vulnerability Mechanism:** Authenticated administrators can upload plugin images through the My Image plugin interface. In vulnerable versions, lack of strict file type extension checks allows uploading raw PHP script files, which are saved in predictable directory locations (e.g., `/nibbleblog/content/private/plugins/my_image/image.php`).

- **Execution:**

1) Leverage an exploit module or upload a custom PHP payload via the My Image plugin administrative settings.
2) Access the uploaded script path directly via the browser or trigger a reverse connection.
3) A successful shell connection is established under the user context of `nibbler`, enabling retrieval of `user.txt`.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Checking assigned `sudo` execution rights for user `nibbler` via `sudo -l` reveals a specific administrative permission:

```
User nibbler may run the following commands on nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

### Missing Directory & Script Hijacking
Inspecting the file path `/home/nibbler/personal/stuff/monitor.sh` reveals that the directory structure (`personal/stuff/`) and the target script do not currently exist on the filesystem.

- **Vulnerability Mechanism:** When a sudo rule points to a specific script path that does not exist, but parent directories or the destination path are writable by the unprivileged user, an attacker can recreate the missing directories and place a malicious script with the exact expected name. When executed via sudo, it runs with root privileges.

- **Privilege Escalation Workflow:**

1) Create the missing directory structure within the user home directory:

```mkdir -p /home/nibbler/personal/stuff/```

2) Create a malicious `monitor.sh` script configured to grant elevated privileges (such as setting the SUID bit on the bash binary or spawning a root shell):

```
#!/bin/bash
chmod u+s /bin/bash
```

3) Assign execution permissions to the script:

```chmod +x /home/nibbler/personal/stuff/monitor.sh```

3) Execute the script via `sudo`:

```sudo -u root /home/nibbler/personal/stuff/monitor.sh```

4) Spawn an interactive privileged shell using the modified binary:

```bash -p```

Full administrative `root` access is acquired, allowing retrieval of root.txt from `/root/root.txt`.
