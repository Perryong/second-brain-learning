---
title: "Cybersecurity Writeups Index"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - writeups
  - hackthebox
  - tryhackme
  - ctf
  - overthewire
  - pwned-labs
  - cryptohack
  - oscp
  - index
---

# Cybersecurity Writeups Index

**Purpose:** Map all personal writeups to relevant PWK modules and techniques. Writeups live in `03-Resources/Practice-Walkthroughs/Writeups/`. All links below open the full writeup.

---

## HackTheBox — Machines

### Active Directory / Windows Domain

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Cicada]] | SMB null session → user enum → AS-REP Roasting → DCSync | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] · [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[EscapeTwo]] | SMB → MSSQL xp_cmdshell → AD enumeration → credential chain | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] · [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[Puppy]] | NXC → SMB enumeration → AD attack path | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] |
| [[Fluffy]] | NXC → SMB → LDAP enumeration → AD exploitation | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] · [[PWK-Module-23-Lateral-Movement-AD\|Mod 23]] |
| [[Certificate]] | ADCS certificate abuse → privilege escalation | [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[Mist]] | HFS exploit → AD enumeration → lateral movement | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] · [[PWK-Module-23-Lateral-Movement-AD\|Mod 23]] |
| [[Jab]] | XMPP enumeration → Openfire CVE → AD (Kerberos) → DCSync | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] · [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[BoardLight]] | Dolibarr CMS exploit → SUID ESC14 privesc (ADCS) | [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] · [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[Crafty]] | Minecraft Log4j → JNDI → Windows foothold → AD creds | [[PWK-Module-24-Assembling-the-Pieces\|Mod 24]] |
| [[Pov]] | SSRF → ViewState deserialization → SeDebugPrivilege token abuse | [[PWK-Module-17-Windows-PrivEsc\|Mod 17]] |
| [[SolarLab]] | ReportHub SSTS → OpenFire CVE-2023-32315 → Mimikatz | [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[Hospital]] | PHP file upload bypass → CVE-2023-35001 kernel LPE → Windows SMTP → ESC1 | [[PWK-Module-17-Windows-PrivEsc\|Mod 17]] · [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |

### Web Application Attacks

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Cap]] | IDOR (PCAP file) → SUID capabilities (cap_setuid) privesc | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Codify]] | CVE-2023-30547 (vm2 sandbox escape) → bash script credential injection | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[CozyHosting]] | SSTI in Spring Boot → command injection → hash crack → sudo privesc | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[Headless]] | XSS cookie theft → SSTI → RCE → sudo privesc | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] · [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[IClean]] | XSS reflected → SSTI (Jinja2) → hash crack → sudo | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[Perfection]] | SSTI (ERB Ruby) → hashcat mask attack on password | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[FormulaX]] | XSS → SSRF → SSTI → command injection chain | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] · [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[Cat]] | XSS → SQLi → hash crack → Apache mod_proxy privesc | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] · [[PWK-Module-10-SQL-Injection\|Mod 10]] |
| [[Kitty]] (THM) | SQLi → credential extraction → privesc | [[PWK-Module-10-SQL-Injection\|Mod 10]] |
| [[Editorial]] | SSRF (internal API discovery) → git history → privesc | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[MonitorsFour]] | CVE-2024-25641 (Cacti RCE) → Docker escape → MariaDB | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Monitored]] | Nagios SNMP → CVE-2023-40931 SQLi → API key → sudo | [[PWK-Module-10-SQL-Injection\|Mod 10]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Sea]] | WonderCMS CVE-2023-41425 (XSS → RCE) → hashcat | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[Usage]] | Laravel SQLi → file upload bypass → Wildcard privesc | [[PWK-Module-10-SQL-Injection\|Mod 10]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Sightless]] | Froxlor SQLite injection → Chrome remote debugging → John crack | [[PWK-Module-10-SQL-Injection\|Mod 10]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[Dog]] | Backdrop CMS CVE → sudo git bootstrap privesc | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Titanic]] | Gitea CVE-2024-42179 (path traversal) → ImageMagick LPE | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |
| [[LinkVortex]] | Ghost CMS CVE-2023-40028 (path traversal) → symlink privesc | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |
| [[BigBang]] | Grafana CVE-2024-9264 (DuckDB RCE) → Wordpress plugin → BuddyPress | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |
| [[PermX]] | Chamilo LMS CVE-2023-4220 (file upload) → symlink ACL abuse | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Devvortex]] | Joomla CVE-2023-23752 (info disclosure) → JohnTheRipper | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[Analytics]] | Metabase CVE-2023-38646 (SSRF+RCE) → Docker env creds | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |
| [[Bizness]] | Apache OFBiz CVE-2023-49070 (auth bypass+RCE) → SHA hash crack | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[GreenHorn]] | Pluck CMS → password in git → Depix pixelated password recovery | [[PWK-Module-16-Password-Attacks\|Mod 16]] |
| [[Mailing]] | CVE-2024-21413 (Outlook NTLM leak) → CVE-2023-2255 (LibreOffice) → WinRM | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-23-Lateral-Movement-AD\|Mod 23]] |
| [[Skyfall]] | CVE-2023-28432 (MinIO info disclosure) → Vault unsealing → metaboss privesc | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |

### Privilege Escalation Focus

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Code]] | Python sandbox escape → HMAC brute force → Gitea → pip privesc | [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Cypher]] | Cypher injection (Neo4j) → BBOT privesc | [[PWK-Module-10-SQL-Injection\|Mod 10]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Kobold]] | C2 framework analysis → command execution | [[PWK-Module-17-Windows-PrivEsc\|Mod 17]] |
| [[Facts]] | PowerShell credential leak → AD enumeration | [[PWK-Module-17-Windows-PrivEsc\|Mod 17]] |
| [[Giveback]] | AWS credential exposure → cloud privesc | [[PWK-Module-17-Windows-PrivEsc\|Mod 17]] |
| [[Pterodactyl]] | Pterodactyl panel → Docker container escape | [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[WingData]] | Cloud/storage misconfiguration | [[PWK-Module-17-Windows-PrivEsc\|Mod 17]] |

### Network / Pivoting

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[WifineticTwo]] | OpenWRT WiFi → WPS Pixie-Dust attack → pivot to internal network | [[PWK-Module-19-Port-Redirection-SSH-Tunneling\|Mod 19]] |
| [[Backfire]] | Havoc C2 SSRF → SSH key injection → iptables privesc | [[PWK-Module-19-Port-Redirection-SSH-Tunneling\|Mod 19]] · [[PWK-Module-20-Metasploit-Framework\|Mod 20]] |

### Miscellaneous / Forensics

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Intuition]] | Werkzeug debugger → RSS SSRF → Suricata rule abuse | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[Headless]] | XSS cookie theft → SSTI RCE | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] |

---

## HackTheBox — Challenges

| Writeup | Category | Key Techniques |
|---------|----------|----------------|
| [[ApacheBlaze]] | Web | CVE Apache HTTP Server exploit |
| [[Behind the Scenes]] | Reversing | Binary analysis, function hooking |
| [[Debugging_Interface]] | Hardware | UART debugging, firmware extraction |
| [[Low Logic]] | Reversing | Assembly analysis |
| [[OnlyHacks]] | Web | Web exploitation |
| [[Photon_Lockdown]] | Hardware | Firmware analysis |
| [[Prometheon]] | Web | Web challenge |
| [[ReactOOPS]] | Web | React XSS / prototype pollution |
| [[Saturn]] | Web | Web exploitation |
| [[Spookifier]] | Web | SSTI exploitation |

---

## HackTheBox — Pro Labs

| Writeup | Coverage |
|---------|----------|
| [[FullHouse]] | Pro lab multi-machine scenario |
| [[Mythical]] | Pro lab multi-machine scenario |
| [[Solar]] | Pro lab — Log4Shell exploitation chain |

---

## TryHackMe — Writeups

### Active Directory

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Attacktive Directory]] | Kerbrute user enum → AS-REP Roasting → Kerberoasting → evil-winrm → Mimikatz | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] · [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |
| [[Soupedecode 01]] | AD enumeration and attack chain | [[PWK-Module-21-Active-Directory-Enumeration\|Mod 21]] |
| [[Reset]] | AD password reset exploitation | [[PWK-Module-22-Attacking-AD-Authentication\|Mod 22]] |

### Web Application Attacks

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Corridor]] | IDOR via MD5 hash manipulation | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[The Sticker Shop]] | Blind XSS → cookie theft | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] |
| [[Kitty]] | SQLi → shell → privesc | [[PWK-Module-10-SQL-Injection\|Mod 10]] |
| [[Neighbour]] | IDOR → account takeover | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[Grep]] | OSINT → API key exposure | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[GeoServer_ CVE-2025-58360]] | GeoServer RCE exploit (CVE-2025-58360) | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |

### Linux / General Pentesting

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[All in One]] | WordPress enumeration → LFI → privesc chain | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[LazyAdmin]] | SweetRice CMS → credential reuse → sudo perl | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Mr Robot CTF]] | WordPress brute-force → webshell → nmap SUID | [[PWK-Module-16-Password-Attacks\|Mod 16]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Pickle Rick]] | Command injection → webshell → root | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |
| [[Overpass]] | JWT manipulation → cron job hijack | [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Simple CTF]] | vBulletin SQLi → FTP → sudo vim | [[PWK-Module-10-SQL-Injection\|Mod 10]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Basic Pentesting]] | SMB enum → SSH brute → sudo privesc | [[PWK-Module-16-Password-Attacks\|Mod 16]] · [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[Cheese CTF]] | PHP type juggling → LFI → SSH key | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] |
| [[Compiled]] | Gitea CVE → privesc | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |
| [[Operation Slither]] | Network analysis → exploitation | [[PWK-Module-19-Port-Redirection-SSH-Tunneling\|Mod 19]] |
| [[WhyHackMe]] | Web enum → privesc chain | [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[TakeOver]] | Subdomain takeover → credential access | [[PWK-Module-09-Common-Web-App-Attacks\|Mod 9]] |

### Windows / Malware

| Writeup | Key Techniques | PWK Module |
|---------|---------------|------------|
| [[Brains]] | WordPress exploit → PHP webshell | [[PWK-Module-08-Web-App-Attacks\|Mod 8]] |
| [[Umbrella]] | Docker exposed → RCE → Node.js privesc | [[PWK-Module-18-Linux-PrivEsc\|Mod 18]] |
| [[TryHack3M_ Bricks Heist]] | WordPress Bricks Builder CVE → privesc | [[PWK-Module-13-Locating-Public-Exploits\|Mod 13]] |

### Misc / CTF-style

| Writeup | Key Techniques | |
|---------|---------------|--|
| [[Vulnerability Capstone]] | Exploit chain combining multiple techniques | Capstone |
| [[The Game]] | Multi-step CTF challenge | CTF skills |
| [[W1seGuy]] | XOR cipher / crypto challenge | Crypto |
| [[c4ptur3-th3-fl4g]] | Encoding/decoding (base64, hex, rot13) | Fundamentals |
| [[Missing Person]] | OSINT / image forensics | Recon |
| [[Cat Pictures]] | FTP anon → IRC creds → sudo docker | Fundamentals |

---

## OverTheWire — Wargames

| Writeup | Coverage | Skills Built |
|---------|----------|-------------|
| [[Bandit]] | Levels 0–30+ SSH wargame | Linux CLI mastery, file permissions, cronjobs, SetUID, scripting, networking |
| [[Natas]] | Web wargame Levels 0–30+ | Web fundamentals: source review, SQLi, LFI, PHP type juggling, brute-force |

---

## CryptoHack

| Writeup | Coverage | PWK Module |
|---------|----------|------------|
| [[Introduction to CryptoHack]] | XOR, modular arithmetic, RSA basics | [[PWK-Module-16-Password-Attacks\|Mod 16]] (hash theory) |
| [[Symmetric Cryptography]] | AES ECB/CBC, block cipher attacks, padding | [[PWK-Module-16-Password-Attacks\|Mod 16]] |

---

## PwnedLabs — Cloud Security

| Writeup | Coverage | Skills Built |
|---------|----------|-------------|
| [[Breach in the Cloud]] | Cloud breach scenario — AWS IAM privilege escalation | [[AWS-SAA-01-IAM-and-Security]] |
| [[Identify the AWS Account ID from a Public S3 Bucket]] | S3 bucket enumeration → account ID disclosure | [[AWS-SAA-03-Storage]] |

---

## CTF Events

### Competitive CTF Writeups

| Event | Writeup | Categories Covered |
|-------|---------|-------------------|
| Cyber Apocalypse 2024 | [[Cyber Apocalypse 2024_ Hacker Royale]] | Web, Crypto, Forensics, Reversing, PWN |
| Hack The Boo 2024 | [[Hack The Boo 2024]] | Web (XSS, SSRF, LFI), Crypto |
| BackdoorCTF 2023 | [[BackdoorCTF_2023]] | Web exploitation, Crypto |
| UofTCTF 2024 | [[UofTCTF_2024]] | Web, Crypto, Reversing |
| UofTCTF 2025 | [[UofTCTF_2025]] | Web, AWS/Cloud, Misc |
| NahamCon CTF | [[NahamCon CTF]] | Web, Misc |
| LA CTF | [[LA CTF]] | Web, Crypto |
| KnightCTF 2024 | [[KnightCTF_2024]] | Web (SQLi, XSS), Crypto |
| University CTF 2024 | [[University CTF 2024_ Binary Badlands]] | Binary exploitation, Reversing |
| NAURYZ CTF 2024 | [[NAURYZ CTF-2024]] | Web, Crypto, AWS |
| Cybercoliseum III | [[Cybercoliseum Ⅲ]] | Web (LFI, SSRF, RCE) |
| IT-FEST CTF 2025 | [[IT-FEST CTF 2025]] | Web |
| BCC CTF 2025 | [[BCC CTF 2025]] | Web, PrivEsc |
| aupCTF | [[aupCTF_(WEB)]] | Web exploitation |
| STS CTF | [[STS CTF]] | Multiple categories |
| KubanCTF Qualifier | [[KubanCTF Qualifier 2024]] | Web, Crypto |
| Mārtiņa CTF 2025 | [[Mārtiņa-CTF-2025]] | Web (LFI, XSS, SQLi) |
| AITU Military CTF | [[AITU Military CTF_ Digital Fortress]] | Web, Crypto |
| TaipanByte CTF | [[TaipanByte CTF]] | Multiple |

---

## Technique Index (Quick Lookup)

Use this to find writeups that practice a specific technique:

| Technique | Writeups |
|-----------|---------|
| **AS-REP Roasting** | [[Cicada]], [[Attacktive Directory]] |
| **Kerberoasting** | [[Attacktive Directory]], [[EscapeTwo]] |
| **DCSync** | [[Cicada]], [[Jab]], [[SolarLab]] |
| **ADCS / Certificate Abuse** | [[Certificate]], [[Hospital]], [[BoardLight]] |
| **MSSQL xp_cmdshell** | [[EscapeTwo]], [[Archetype]] |
| **SSTI (Jinja2/ERB/Twig)** | [[IClean]], [[Perfection]], [[CozyHosting]], [[FormulaX]], [[Spookifier]] |
| **XSS → Escalation** | [[Headless]], [[Cat]], [[FormulaX]], [[The Sticker Shop]] |
| **SSRF** | [[Editorial]], [[Analytics]], [[Backfire]], [[Monitored]] |
| **File Upload Bypass** | [[Hospital]], [[PermX]], [[Usage]] |
| **CVE Exploitation** | [[Bizness]], [[Analytics]], [[Devvortex]], [[Sea]], [[Mailing]], [[Codify]], [[PermX]] |
| **Docker Escape** | [[Analytics]], [[MonitorsFour]], [[Pterodactyl]], [[Umbrella]] |
| **SUID / Capabilities** | [[Cap]], [[BoardLight]], [[Mr Robot CTF]] |
| **Sudo Abuse** | [[Codify]], [[CozyHosting]], [[Code]], [[Simple CTF]] |
| **Git History Creds** | [[Editorial]], [[GreenHorn]], [[Titanic]] |
| **Hash Cracking** | [[IClean]], [[Perfection]], [[Bizness]], [[GreenHorn]], [[Devvortex]] |
| **WiFi / WPS** | [[WifineticTwo]] |
| **Log4Shell** | [[Crafty]], [[Solar]] |
| **SQLi (various)** | [[Monitored]], [[Usage]], [[Cypher]], [[Cat]], [[Kitty]], [[Simple CTF]] |
| **SSH Key Abuse** | [[Backfire]], [[Cheese CTF]], [[Overpass]] |
| **JWT Manipulation** | [[Overpass]] |
| **Cron Job Hijack** | [[Overpass]], [[Mr Robot CTF]] |
| **AWS Cloud Attacks** | [[Giveback]], [[PwnedLabs - Breach in the Cloud\|Breach in the Cloud]], [[Identify the AWS Account ID from a Public S3 Bucket]] |
| **Linux Wargame (SSH)** | [[Bandit]] |
| **Web Wargame** | [[Natas]] |
| **Crypto / Encoding** | [[Introduction to CryptoHack]], [[Symmetric Cryptography]], [[W1seGuy]] |

---

## Connections

- [[THM-HTB-Practice-Labs-Index]] — older practice lab index (aligned to PWK modules)
- [[PWK-Module-08-Web-App-Attacks]] · [[PWK-Module-09-Common-Web-App-Attacks]] · [[PWK-Module-10-SQL-Injection]]
- [[PWK-Module-13-Locating-Public-Exploits]] · [[PWK-Module-16-Password-Attacks]]
- [[PWK-Module-17-Windows-PrivEsc]] · [[PWK-Module-18-Linux-PrivEsc]]
- [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] · [[PWK-Module-20-Metasploit-Framework]]
- [[PWK-Module-21-Active-Directory-Enumeration]] · [[PWK-Module-22-Attacking-AD-Authentication]]
- [[PWK-Module-23-Lateral-Movement-AD]] · [[PWK-Module-24-Assembling-the-Pieces]]
- [[AWS-SAA-01-IAM-and-Security]] · [[AWS-SAA-03-Storage]]
