# Pilgrimage

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.78.201`

## Summary
**Pilgrimage** is an easy-difficulty Linux machine that illustrates risks associated with exposed version control directories, insecure image processing binaries, and automated administrative scanning utilities. Initial access is achieved by identifying an exposed `.git/` repository via directory enumeration, dumping the source code, and exploiting an Arbitrary File Read vulnerability (**CVE-2022-44268**) in an outdated version of **ImageMagick (7.1.0-49)** to extract SQLite database credentials. Subsequent SSH authentication grants access as user `emily`. Privilege escalation to root is accomplished by identifying a cron/system script (`malwarescan.sh`) that periodically executes an insecure version of Binwalk (v2.3.2), allowing remote code execution via a crafted image file processed with root privileges.

## Reconnaissance

### Port Scanning (Nmap)
We begin by enumerating active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 8.4p1 (Debian 5+deb11u1))
- **80**: HTTP (nginx 1.18.0)

### Web Application & Git Repository Exposure
1) Visiting the web application on port 80 reveals an image compression and resizing utility featuring user registration and login functionality.
2) Enumerating web files or running vulnerability scan scripts (`http-enum`) uncovers an exposed `.git` directory (`/.git/HEAD`).
3) Using `git-dumper`, we successfully reconstruct the entire web application source code repository locally for static code analysis:

```
python3 git_dumper.py http://pilgrimage.htb/ pilgrimage
```

## Exploitation (Initial Access)

### ImageMagick Arbitrary File Read (CVE-2022-44268)
Analyzing the dumped source code reveals that the application handles image compression using a local binary (magick, version 7.1.0-49).

- **Vulnerability Mechanism:** ImageMagick versions prior to 7.1.0-51 contain an arbitrary file read vulnerability when handling specific image formats (such as Portable Network Graphics / PNG containing crafted text chunks or `EXIF` data referencing external file pointers via pseudo-protocols).
- **Execution:**
  1) Crafting a malicious image payload that forces ImageMagick to read a targeted system file (e.g., `/etc/passwd` or the SQLite database file containing application credentials).
  2) Uploading the crafted image through the web interface. The server processes the image using the vulnerable binary and provides a resized link.
  3) Downloading the processed image and parsing its binary content (using `identify -verbose` or extraction scripts) reveals the contents of the target file.
  4) Reading the application's SQLite database file located at `/var/lib/pilgrimagefiles/pilgrimage.sqlite` exposes stored user accounts and password hashes. Decoding the hash or recovering cleartext credentials reveals the password for user `emily`:

Credentials: `emily` : `abigchonkyboi123`

### SSH Authentication
Using the recovered credentials, we authenticate via SSH:

```
ssh emily@10.129.78.201
```

We obtain initial user shell access and retrieve `user.txt`.

## Privilege Escalation (Root Flag)

### Process & Background Task Enumeration
Checking active system processes via `ps -faux` reveals a recurring administrative script executed periodically under the `root` context:

```
root         756  0.0  0.0   6816  2308 ?        S    00:35   0:00  \_ /bin/bash /usr/sbin/malwarescan.sh
```

Inspecting the contents of `/usr/sbin/malwarescan.sh`:

```
#!/bin/bash
blacklist=("Executable script" "Microsoft executable")
/usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
	filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
	binout="$(/usr/local/bin/binwalk -e "$filename")"
        for banned in "${blacklist[@]}"; do
		if [[ "$binout" == *"$banned"* ]]; then
			/usr/bin/rm "$filename"
			break
		fi
	done
done
```

**Vulnerability Analysis:** The script monitors the image output directory using inotifywait and automatically passes newly created files into Binwalk (`/usr/local/bin/binwalk`) with the extraction flag (`-e`) running as `root`.

### Binwalk Remote Code Execution (CVE-2022-4510)
The installed Binwalk version is v2.3.2, which contains a well-known Remote Code Execution vulnerability in how it handles compressed archives (such as tar or zip metadata inside crafted image files).

- **Exploitation Workflow:**

1) Utilizing a public exploit targeting CVE-2022-4510 (e.g., `WalkingPath`), we generate a crafted image containing a malicious payload designed to write an authorized SSH key or a reverse shell script into `/root/` when parsed by Binwalk.
2) Uploading or placing the crafted image into `/var/www/pilgrimage.htb/shrunk/` triggers the `malwarescan.sh` monitoring script.
3) When Binwalk extracts the file with root privileges, the payload executes automatically.
4) Accessing the system via the established root connection or SSH key grants full administrative control.

Retrieving `root.txt` from `/root/root.txt` completes the machine compromise.
