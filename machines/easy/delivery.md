# Delivery

**Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.54.75`

## Summary
**Delivery** is an easy-difficulty Linux machine featuring an internal Helpdesk system and a Mattermost messaging server. Initial access is achieved by utilizing the Helpdesk ticket platform (`helpdesk.delivery.htb`) to generate a temporary `@delivery.htb` email address, which is then used to register and activate an account on the Mattermost instance running on port 8065. Inside Mattermost, internal credentials for `maildeliverer` are recovered. Privilege escalation to root is accomplished by extracting database credentials from Mattermost's `config.json`, dumping the `Users` table from MySQL, and performing a rule-based password cracking attack with `hashcat` using the `best64.rule` ruleset to target the root account's bcrypt hash.

## Reconnaissance

### Port Scanning (Nmap)
We begin by scanning active TCP ports and identifying service versions:

- **22**: SSH (OpenSSH 7.9p1 Debian 10)
- **80**: HTTP (nginx 1.14.2)
- **8065**: HTTP (Golang net/http server (Mattermost) )

### Web Enumeration & Domain Mapping
Inspecting port 80 reveals a contact page linking to a OsTicket portal hosted at `helpdesk.delivery.htb`. Port 8065 hosts a Mattermost enterprise team messaging server. We update our local `/etc/hosts` file accordingly:

```echo "10.129.54.75 delivery.htb helpdesk.delivery.htb" >> /etc/hosts```

Directory enumeration reveals standard web directories (`/assets`, `/images`, `/error`).

## Exploitation (Initial Access)

### Helpdesk Email Registration & Mattermost Access

1) We open a new support ticket on `helpdesk.delivery.htb`. The system assigns a unique Ticket ID and displays a temporary ticket email address under the `@delivery.htb` domain (e.g., `[ticket_id]@delivery.htb`).
2) We attempt to sign up for a new account on the Mattermost portal (`[http://delivery.htb:8065]`) using the newly acquired `@delivery.htb` ticket email.
3) Mattermost dispatches a verification email internally to the ticket mailbox.
4) We check the status of our ticket on `helpdesk.delivery.htb`, retrieve the verification link sent by Mattermost, and confirm account registration.

### Credential Extraction & SSH Access

Loging into Mattermost and viewing internal channels reveals exposed system credentials intended for developer onboarding:

Extracted Credentials: `maildeliverer` : `Youve_G0t_Mail!`

We authenticate as maildeliverer via SSH to secure initial access:

```ssh maildeliverer@10.129.54.75```

We retrieve user.txt from `/home/maildeliverer/user.txt`.

## Privilege Escalation (Root Flag)

### Configuration Enumeration & MySQL Access
Enumerating active system processes reveals the Mattermost installation directory:

```ps -faux | grep -i mattermost```

Inspecting the configuration file at `/opt/mattermost/config/config.json` yields local MySQL database credentials:

```"DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s"```

We connect to the local MySQL server using `mmuser`:

```mysql -u mmuser -p'Crack_The_MM_Admin_PW' -h 127.0.0.1 mattermost```

We query the `Users` table to extract account user names and bcrypt hashes:

```SELECT username, password FROM Users;```

```
+----------------------------------+--------------------------------------------------------------+
| username                         | password                                                     |
+----------------------------------+--------------------------------------------------------------+
| root                             | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO |
+----------------------------------+--------------------------------------------------------------+
```

### Rule-Based Hash Cracking with Hashcat
Standard dictionary attacks against the `$2a$` bcrypt hash using `rockyou.txt` fail. However, internal chat logs inside Mattermost referenced a password variation rule based on the phrase `PleaseSubscribe!`.

We generate a targeted wordlist by applying the best64.rule ruleset against a base file containing `PleaseSubscribe!`:

```
echo "PleaseSubscribe!" > base
hashcat --stdout base -r /usr/share/hashcat/rules/best64.rule > custom_dict.txt
```

We execute `hashcat` targeting the bcrypt hash mode (`3200`):

```hashcat -m 3200 -a 0 root_hash.txt custom_dict.txt```

Recovered Root Password: `PleaseSubscribe!21`

We switch to the root account using `su`

We achieve full `root` privileges and capture `root.txt`.
