# Editorial

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.56.91`

## Summary
**Editorial** is an easy-difficulty Linux target that illustrates risks associated with Server-Side Request Forgery (SSRF) and Git repository history exposures. Initial access is achieved by identifying an SSRF vulnerability in the web application's cover preview feature, which allows probing internal services. Querying an unexposed internal API running on port 5000 leaks plaintext SSH credentials for user `dev`. Privilege escalation is completed in two stages: first, inspecting the local Git history reveals hardcoded credentials for user `prod`; second, leveraging a GitPython vulnerability (CVE-2022-24439) in an administrative script executable via sudo grants full `root` access.

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **22**: SSH (OpenSSH 8.9p1 (Ubuntu 3ubuntu0.7))
- **80**: HTTP (nginx 1.18.0)

### Directory & Web Enumeration
Enumerating the web application reveals a publishing platform with key routes:

- `/about` — General company overview.
- `/upload` — Submission form allowing authors to submit draft content and specify an external cover image URL for preview.

## Exploitation (Initial Access)

### Server-Side Request Forgery (SSRF) Analysis
The preview feature under `/upload` accepts a remote image URL, fetches the resource on the server side, saves it locally under `static/uploads/`, and renders it back to the client.

- **Vulnerability Mechanism:** Lack of input validation on the URL parameter allows requesting internal loopback addresses (http://127.0.0.1).
- **Internal Service Probing:** Fuzzing local ports via the SSRF vulnerability reveals an unauthenticated internal API listening locally on port `5000`.

### API Information Leakage & SSH Access
Interacting with the internal API endpoint `http://127.0.0.1:5000/api/latest/metadata` exposes several documentation paths. Querying the author onboarding endpoint reveals sensitive credentials embedded in email templates:

**Internal Endpoint:** `/api/latest/metadata/messages/authors`
**Recovered Credentials:** `dev` : `dev080217_devAPI!@`

With SSH open on port 22, we log in using the discovered credentials:

```ssh dev@10.129.56.91```

We gain initial shell access as `dev` and retrieve `user.txt`.

## Privilege Escalation (Root Flag)

### Lateral Movement: Git Repository Analysis
Inspecting user directories reveals local Git repositories under `/apps`. Reviewing historical commit logs identifies a commit titled `change(api): downgrading prod to dev`.

- **Information Exposure:** Diffing historical commits reveals hardcoded credentials previously used for production accounts.
- **Recovered Credentials:** `prod` : `080217_Producti0n_2023!@`

We pivot to the `prod` user via `su prod` or SSH.

### Privilege Escalation: GitPython Vulnerability (CVE-2022-24439)
Checking Sudo privileges for `prod` (`sudo -l`) shows execution rights for a Python utility as `root`:

```(root) /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *```

Analyzing `clone_prod_change.py` indicates it utilizes the GitPython library to handle repository cloning actions.

- **Vulnerability Mechanism (CVE-2022-24439):** Older versions of GitPython pass untrusted URL inputs directly to underlying `git` commands without proper sanitization. Protocol extensions like `ext::` allow executing arbitrary system commands during repository initialization.
- **Impact:** Passing a crafted protocol string to the script causes GitPython to execute arbitrary commands with root privileges.

Executing the script with command injection arguments yields an elevated shell and enables retrieval of `root.txt`.
