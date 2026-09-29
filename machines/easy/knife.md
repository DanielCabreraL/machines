# Knife

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.62.173`

## Summary
**Knife** is an easy-difficulty Linux target that highlights the severe risks of supply chain compromises in web development software and misplaced `sudo` privileges. Initial access is achieved by identifying a compromised version of the PHP interpreter (**PHP 8.1.0-dev**) containing an unauthenticated remote code execution backdoor embedded in HTTP headers. Privilege escalation to `root` is performed by exploiting sudo execution rights assigned to the Chef Knife administrative binary (`/usr/bin/knife`), leveraging native Ruby execution options to spawn an elevated shell.

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **22**: SSH (OpenSSH 8.2p1 (Ubuntu 4ubuntu0.2))
- **80**: HTTP (Apache httpd 2.4.41)

### Web Application & Version Enumeration
Inspecting HTTP response headers returned by the Apache web server reveals the underlying script engine version:

```X-Powered-By: PHP/8.1.0-dev```

## Exploitation (Initial Access)

### PHP 8.1.0-dev Backdoor Execution (RCE)
In March 2021, the official PHP git repository was compromised, and a backdoor was inserted into the `PHP 8.1.0-dev` development branch.

- **Vulnerability Mechanism:** The malicious commit added code to check for the presence of a custom HTTP request header string beginning with `zerodium`. If present, the server executes the arbitrary string specified after `zerodium` as system PHP code via zend_eval_string.

- **Execution:**
Sending a request containing the specific `User-Agentt` header variant (e.g., `User-Agentt: zerodiumvar_dump(1);`) triggers instant code execution under the context of the james account.

- **Interactive Access:**
Executing an interactive reverse shell payload via the header grants terminal access on the host as user `james`, enabling retrieval of `user.txt`.

## Privilege Escalation (Root Flag)

### Sudo Permission Enumeration
Inspecting assigned `sudo` rights for user `james` via `sudo -l` reveals elevated execution privileges without requiring a password:

```
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

### Chef Knife Binary Exploitation
The `/usr/bin/knife` utility is part of the Chef infrastructure automation framework written in Ruby.

- **Vulnerability Mechanism:** The `knife exec` subcommand allows executing arbitrary Ruby code blocks directly from the command line interface using the `-E` (or `--eval`) parameter.
- **Privilege Escalation Workflow:** Passing a Ruby process invocation command (`exec "/bin/sh"`) inside the `knife exec` argument while executing under `sudo` causes the command interpreter to spawn a subshell inheriting `root` privileges:

```sudo knife exec -E 'exec "/bin/sh"'```

Executing the command immediately replaces the process with an interactive `root` shell, granting complete control over the system and access to `root.txt`.
