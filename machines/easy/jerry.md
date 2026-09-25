# Jerry

**Difficulty:** Easy | **OS:** Windows | **IP:** `10.129.136.9`

## Summary
**Jerry** is an easy-difficulty Windows machine that highlights the risks associated with default deployment credentials in web application managers and unconstrained administrative privileges on web application servers. Initial access is achieved by authenticating to the Apache Tomcat Manager interface (`/manager/html`) using default credentials (`tomcat` : `s3cret`). Leveraging Tomcat's administrative features, a standard Java Web Application archive (`.war`) payload is deployed to execute code within the context of the running service. Because the Tomcat service runs under elevated privileges, initial code execution immediately grants full `NT AUTHORITY\SYSTEM` access without requiring local privilege escalation.

## Reconnaissance

### Port Scanning (Nmap)
We begin by identifying open TCP ports and active service versions:

- **8080**: HTTP (Apache Tomcat/Coyote JSP engine 1.1)

### Web Application & Service Enumeration
Navigating to `http://10.129.136.9:8080/` presents the default Apache Tomcat landing page.

1) Directory enumeration identifies the administrative manager interface located at `/manager/html`.
2) Attempting to access `/manager/html` prompts an HTTP Basic Authentication challenge.
3) Submitting common default credentials or inspecting default Tomcat documentation/example configuration notes reveals active administrative credentials:

  - Username: `tomcat`
  - Password: `s3cret`

## Exploitation (Initial Access & SYSTEM Access)

### Tomcat Manager WAR Application Deployment
The Tomcat Web Application Manager allows users with the `manager-script` or `manager-gui` role to deploy packaged Java Web Applications (`.war` files).

- **Vulnerability Mechanism:** Authenticated administrative access to the manager panel permits uploading arbitrary .war files containing executable Java Server Pages (JSP). When requested, Tomcat compiles and executes the uploaded JSP code under the service account context.
- **Execution & Access:**
 
1) A custom `.war` package containing a reverse shell JSP component is created.
2) The application archive (`exploit.war`) is uploaded and deployed via the Manager UI form.
3) Browsing to the newly deployed application endpoint (`http://10.129.136.9:8080/exploit/`) triggers the execution of the JSP payload.
4) An incoming connection is caught on an active network listener.

## Privilege Verification
Inspecting the process execution context using `whoami` confirms the active shell identity:

```
whoami
# nt authority\system
```

Because Apache Tomcat was configured to run under the highest system privilege context (`NT AUTHORITY\SYSTEM`), both user and system administrative flags (`user.txt` and `root.txt`) are accessible immediately from `C:\Users\Administrator\Desktop\flags\`.
