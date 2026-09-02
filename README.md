# 🖥️ Hack The Box - Writeups

![HTB](https://img.shields.io/badge/HackTheBox-111927?style=for-the-badge&logo=hackthebox&logoColor=9FEF00)
![GitHub last commit](https://img.shields.io/github/last-commit/[tu-usuario]/[tu-repo])
![GitHub repo size](https://img.shields.io/github/repo-size/[tu-usuario]/[tu-repo])
![GitHub stars](https://img.shields.io/github/stars/[tu-usuario]/[tu-repo]?style=social)

---

## 📖 About this repository

This repository contains my personal writeups for **retired** machines from [Hack The Box](https://www.hackthebox.com/).

My goal is not just to show the commands used, but to **explain the reasoning behind each step**, documenting the thought process, enumeration techniques, and attack vectors employed. Each writeup is designed to be a learning resource for myself and others in the cybersecurity community.

---

## 🎯 Methodology

Each writeup follows a consistent structure:

1. **Reconnaissance** - Port scanning, service enumeration, and fingerprinting
2. **Deep Enumeration** - Analysis of web services, SMB, DNS, etc.
3. **Exploitation** - Identifying and exploiting vulnerabilities
4. **Privilege Escalation** - Techniques to gain root/system access

---

## 📊 Machine Status

| Machine | OS | Difficulty | Key Techniques | Writeup |
|:--------|:--:|:----------:|:----------------|:--------|
| [Machine_Name] | Linux | Easy | `sqli`, `idor`, `suid` | [View →](./machines/easy/[Machine_Name]/) |
| [Machine_Name] | Windows | Medium | `kerberoasting`, `juicy-potato` | [View →](./machines/medium/[Machine_Name]/) |
| [Machine_Name] | Linux | Hard | `buffer-overflow`, `docker-escape` | [Coming soon] |

> **Note:** All documented machines are **retired** at the time of publication.

---

## 📂 Repository Structure

HackTheBox-Writeups/
├── README.md # This file
├── machines/ # Main directory for machines
│ ├── easy/ # Easy difficulty machines
│ │ └── [Machine_Name]/
│ │ ├── README.md # The writeup itself
│ │ ├── images/ # Screenshots
│ │ └── scripts/ # Custom scripts created
│ ├── medium/
│ ├── hard/
│ └── insane/
└── resources/ # (Optional) Cheatsheets or guides


---

## 🛠️ Tools Used

| Category | Tools |
|:---------|:------|
| **Scanning** | Nmap, masscan |
| **Web Enumeration** | Gobuster, ffuf, dirb, Burp Suite |
| **Exploitation** | Metasploit, custom Python/Bash scripts |
| **Password Cracking** | John the Ripper, Hashcat, hydra |
| **Privilege Escalation** | linPEAS, winPEAS, pspy |
| **Traffic Analysis** | Wireshark, tcpdump |

---

## 🚀 How to use this repository

1. Navigate through the folders by difficulty (`easy/`, `medium/`, `hard/`, `insane/`)
2. Each machine has its own directory with:
   - `README.md` - The complete writeup
   - `images/` - Screenshots of the process
   - `scripts/` - Scripts created for the machine (if applicable)
3. Read the writeup in order to understand the full attack chain

---

## 📈 Progress

| Difficulty | Completed | Total (Approx) |
|:-----------|:---------:|:--------------:|
| 🟢 Easy    | [N]       | [X]            |
| 🟡 Medium  | [N]       | [X]            |
| 🔴 Hard    | [N]       | [X]            |
| ⚫ Insane   | [N]       | [X]            |
| **Total**  | **[N]**   | **[X]**        |

---

## 📚 Inspiration

This repository is inspired by the work of great professionals in the community:

- [momenbasel](https://github.com/momenbasel/htb-writeups) - One of the most comprehensive writeup collections
- [Caan31](https://github.com/Caan31/-HackTheBox-Writeups-by-Arabot) - Excellent explanations and clear formatting
- [0xdf](https://0xdf.gitlab.io/) - Detailed technical blog posts with deep analysis
- [ippsec](https://www.youtube.com/c/ippsec) - Legendary HTB video walkthroughs

---

## 🤝 Contributing

While this is primarily a personal repository, I welcome suggestions and corrections. If you spot an error or have a more efficient approach:

1. Open an Issue explaining the improvement
2. Fork the repository and create a Pull Request
3. Make sure to follow the existing structure and format

Please read the [CONTRIBUTING.md](./CONTRIBUTING.md) file for more details.

---

## ⚠️ Legal Disclaimer

This material is **purely educational**. All content is intended to improve cybersecurity skills in controlled and authorized environments.

- All machines documented are **retired** from Hack The Box
- Flags and sensitive data are **never** published
- The techniques shown should **only** be used on systems you own or have explicit permission to test

---

## 📬 Contact

- **GitHub:** [@[tu-usuario]](https://github.com/[tu-usuario])
- **HTB Profile:** [HTB Profile](https://www.hackthebox.com/users/[tu-id])
- **LinkedIn:** [Tu LinkedIn] (optional)
- **Twitter/X:** [@tu-usuario] (optional)

---

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

---

⭐ **If this repository has been helpful to you, consider giving it a star!**

---

*Happy Hacking! 🚀*
