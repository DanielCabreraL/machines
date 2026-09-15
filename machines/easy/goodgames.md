# GoodGames

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.56.237`

## Summary
**GoodGames** is an easy-difficulty Linux machine demonstrating risks associated with SQL Injection (SQLi), Server-Side Template Injection (SSTI) in Python web frameworks, and shared directory mounts between Docker containers and host systems. Initial access to the administration portal is obtained by exploiting an SQL injection flaw in the primary login form to extract hashed administrator credentials. Once inside the internal admin interface, an SSTI vulnerability allows executing code inside a Docker container. Escalation to the host system and full root access is completed by identifying a shared filesystem volume and creating a SUID binary executable on the host.

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **80**: HTTP (Werkzeug httpd 2.0.2 (Python 3.9.2))

### Directory & Web Enumeration
Fuzzing directories using ffuf reveals standard web routes:

- `/blog` — Main blog section.
- `/login` — User authentication portal.
- `/signup` — Registration page.
- `/profile` — User profile management area.

## Exploitation (Initial Access)

### SQL Injection (SQLi) & Credential Extraction
Testing the main login form (/login) for authentication bypass reveals an SQL injection vulnerability in the email parameter:

- Submitting `test@test.com' OR 1=1-- -` bypasses the initial authentication check, logging the session in as admin.
- To access the internal administration portal (`internal-administration.goodgames.htb`), administrative credentials are required.
- Exploiting the SQL injection flaw (determining column counts via response length analysis) enables extracting database records containing the admin password hash:

Username: `admin`
MD5 Hash: `2b22337f218b2d82dfc3b6f77e7cb8ec`
Cracked Password: `superadministrator`

### Server-Side Template Injection (SSTI)
Logging into the internal administration interface with `admin` : `superadministrator` grants access to the settings management page.

- Injecting standard template syntax `{{7*7}}` into profile settings returns `49`, confirming a Server-Side Template Injection (SSTI) flaw within the Jinja2 rendering engine.
- Executing system commands via Python global scope objects (`cycler.__init__.__globals__.os.popen`) allows remote code execution.
- Spawning an interactive reverse shell connects back to our attack host, granting shell access inside a restricted Docker container.

## Privilege Escalation (Root & Host Escape)

### Container Enumeration & Lateral Movement
Checking environment variables and IP configurations indicates execution inside a isolated container (`172.19.0.2`).

- Scanning the host gateway (`172.19.0.1`) reveals open port 22 (SSH).
- Authenticating via SSH using the local container username (`augustus`) and reusing the password `superadministrator` grants shell access to the host machine.

### Container Escape via Shared Mount & SUID Abuse
Comparing host and container filesystems identifies a shared directory mount point shared between the host and the container instance.

- **Vulnerability Mechanism:** Files created inside the shared volume are immediately visible to both the host and container. If a file's ownership and SUID bit are modified by `root` inside the container (where the process runs as `root`), those file permissions persist on the host filesystem.
- **Privilege Escalation Workflow:**

  1) On the host machine, copy the system `/bin/bash` binary into the shared directory.
  2) Inside the container session (where shell context is `root`), change ownership of the copied binary to `root:root` and assign SUID permissions:

  ```
  chown root:root bash
  chmod 4755 bash
  ```

  3) Return to the host user session (`augustus`) and execute the modified binary with preserved privileges:

     ```./bash -p```

Executing the binary with elevated privileges grants full `root` access on the host system, enabling retrieval of both `user.txt` and `root.txt`.
