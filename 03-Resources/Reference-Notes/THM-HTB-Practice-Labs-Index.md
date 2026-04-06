---
title: "TryHackMe & HackTheBox — Practice Labs Index"
created: "2026-04-06"
type: reference-note
status: active
tags:
  - reference-note
  - tryhackme
  - hackthebox
  - oscp
  - practice-labs
  - index
---

# TryHackMe & HackTheBox — Practice Labs Index

**Purpose:** Map every completed lab in Perry's THM/HTB collection to the relevant PWK module. Use this to find hands-on practice for any topic, or to understand which real-world scenarios correspond to a given module.

---

## Module 8–10 — Web App Attacks / SQL Injection

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Game Zone](https://tryhackme.com/room/gamezone) | TryHackMe | Manual SQLi → sqlmap → hash dump → hashcat → SSH reverse tunnel |
| [Bookstore](https://tryhackme.com/room/bookstoreoc) | TryHackMe | REST API enumeration, fuzzing hidden parameters |
| [NoSQL Injection Basics](https://tryhackme.com/room/nosqlinjectionbasics) | TryHackMe | MongoDB NoSQL injection bypass |
| [Road](https://tryhackme.com/room/road) | TryHackMe | IDOR → file upload → SUID binary PrivEsc |
| [Authentication Bypass](https://tryhackme.com/room/authenticationbypass) | TryHackMe | Username enumeration, brute-force, logic flaws |
| [Command Injection](https://tryhackme.com/room/comminjection) | TryHackMe | OS command injection via web form |
| [Upload Vulnerabilities](https://tryhackme.com/room/uploadvulns) | TryHackMe | File upload bypass (MIME, extension, magic bytes) |
| [Unified](https://app.hackthebox.com/machines/Unified) | HackTheBox | Log4Shell (CVE-2021-44228) → MongoDB cred extraction |

**→ PWK Notes:** [[PWK-Module-08-Web-App-Attacks]] · [[PWK-Module-09-Common-Web-App-Attacks]] · [[PWK-Module-10-SQL-Injection]]

---

## Module 11–12 — Phishing / Client-Side Attacks

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Advent of Cyber 2022](https://tryhackme.com/room/adventofcyber4) | TryHackMe | Phishing analysis, macro-based attacks (Task 3–4) |
| [Brute Force Heroes](https://tryhackme.com/room/bruteforceheroes) | TryHackMe | Credential phishing simulation |

**→ PWK Notes:** [[PWK-Module-11-Phishing-Basics]] · [[PWK-Module-12-Client-Side-Attacks]]

---

## Module 13–14 — Public Exploits / Buffer Overflows

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Buffer Overflow Prep](https://tryhackme.com/room/bufferoverflowprep) | TryHackMe | 10× OVERFLOW exercises — complete x86 Windows BoF methodology (EIP control, bad char, JMP ESP, shellcode) |
| [Buffer Overflows](https://tryhackme.com/room/bof1) | TryHackMe | Basic stack overflow introduction |
| [Brainstorm](https://tryhackme.com/room/brainstorm) | TryHackMe | x86 Windows stack buffer overflow — full exploit development |
| [Brainpan 1](https://tryhackme.com/room/brainpan) | TryHackMe | Linux/Windows BoF with OSCP-style challenge format |

**→ PWK Notes:** [[PWK-Module-13-Locating-Public-Exploits]] · [[PWK-Module-14-Fixing-Exploits]]

---

## Module 16 — Password Attacks

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Password Attacks](https://tryhackme.com/room/passwordattacks) | TryHackMe | CUPP/crunch/cewl wordlist generation, Hydra/Medusa, hashcat modes |
| [Game Zone](https://tryhackme.com/room/gamezone) | TryHackMe | sqlmap hash extraction → hashcat cracking |
| [Capture!](https://tryhackme.com/room/capture) | TryHackMe | Rate-limited brute-force with CAPTCHA bypass |

**→ PWK Notes:** [[PWK-Module-16-Password-Attacks]]

---

## Module 17 — Windows Privilege Escalation

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Windows Privilege Escalation](https://tryhackme.com/room/windowsprivesc20) | TryHackMe | Service misconfigs, AlwaysInstallElevated, scheduled tasks, credential harvesting (IIS/PuTTY/PowerShell history) |
| [Steel Mountain](https://tryhackme.com/room/steelmountain) | TryHackMe | Rejetto HFS CVE-2014-6287 → unquoted service path / insecure binary path PrivEsc |
| [Alfred](https://tryhackme.com/room/alfred) | TryHackMe | Jenkins exploit → Meterpreter → token impersonation (incognito) → SYSTEM |
| [Internal](https://tryhackme.com/room/internal) | TryHackMe | Full internal pentest (WordPress → Jenkins → SSH privkey → root) |
| [Abusing Windows Internals](https://tryhackme.com/room/abusingwindowsinternals) | TryHackMe | DLL injection, process hollowing, token manipulation |
| [Windows Local Persistence](https://tryhackme.com/room/windowslocalpersistence) | TryHackMe | Registry autorun, backdoor service, RDP backdoor techniques |

**→ PWK Notes:** [[PWK-Module-17-Windows-PrivEsc]]

---

## Module 18 — Linux Privilege Escalation

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Common Linux Privesc](https://tryhackme.com/room/commonlinuxprivesc) | TryHackMe | SUID, sudo misconfigs, writable /etc/passwd, cron jobs, path hijacking |
| [Linux Local Enumeration](https://tryhackme.com/room/linuxlocalenumeration) | TryHackMe | Systematic local enumeration methodology |
| [0day](https://tryhackme.com/room/0day) | TryHackMe | Shellshock (CVE-2014-6271) → Linux kernel PrivEsc (overlayfs) |
| [Internal](https://tryhackme.com/room/internal) | TryHackMe | SSH private key discovery after service pivoting |
| [Oh My WebServer](https://tryhackme.com/room/ohmyweb) | TryHackMe | CVE-2021-41773 Apache RCE → Docker escape → root |

**→ PWK Notes:** [[PWK-Module-18-Linux-PrivEsc]]

---

## Module 19 — Port Redirection and SSH Tunneling

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Wreath](https://tryhackme.com/room/wreath) | TryHackMe | Full multi-hop pivoting course: SSH -L/-R, Socat, Chisel, sshuttle, plink.exe across 3-machine network |
| [Extending Your Network](https://tryhackme.com/room/extendingyournetwork) | TryHackMe | Port forwarding and firewall fundamentals |
| [Game Zone](https://tryhackme.com/room/gamezone) | TryHackMe | SSH reverse tunnel (-R) to expose internal Webmin to Kali |
| [Badbyte](https://tryhackme.com/room/badbyte) | TryHackMe | SSH tunneling to reach services on a segmented internal network |

**→ PWK Notes:** [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] · [[Pivoting-Mental-Model]]

---

## Module 20 — The Metasploit Framework

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Metasploit](https://tryhackme.com/room/metasploitintro) | TryHackMe | msfdb init, msfconsole basics, auxiliary modules, MS17-010 (EternalBlue) |
| [Metasploit: Exploitation](https://tryhackme.com/room/metasploitexploitation) | TryHackMe | db_nmap integration, vulnerability scanning, msfvenom payload generation |
| [Metasploit: Meterpreter](https://tryhackme.com/room/meterpreter) | TryHackMe | Meterpreter in-memory architecture, process migration, channel commands |
| [Alfred](https://tryhackme.com/room/alfred) | TryHackMe | Jenkins → Meterpreter → getsystem via token impersonation |
| [AV Evasion: Shellcode](https://tryhackme.com/room/avevasionshellcode) | TryHackMe | msfvenom shellcode encoding/packing to evade AV detection |

**→ PWK Notes:** [[PWK-Module-20-Metasploit-Framework]] · [[Metasploit-Mental-Model]]

---

## Module 21 — Active Directory Enumeration

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Active Directory Basics](https://tryhackme.com/room/activedirectorybasics) | TryHackMe | Domain structure, users/groups/OUs, GPOs, trusts |
| [Attacking Kerberos](https://tryhackme.com/room/attackingkerberos) | TryHackMe | Kerbrute user enumeration, SPN discovery, BloodHound context |
| [VulnNet: Roasted](https://tryhackme.com/room/vulnnetroasted) | TryHackMe | Null session SMB → share enumeration → user discovery → credential attacks |
| [RazorBlack](https://tryhackme.com/room/raz0rblack) | TryHackMe | AD enumeration pipeline leading into authentication attacks |

**→ PWK Notes:** [[PWK-Module-21-Active-Directory-Enumeration]] · [[AD-Enumeration-to-Attack-Path]]

---

## Module 22 — Attacking Active Directory Authentication

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Attacking Kerberos](https://tryhackme.com/room/attackingkerberos) | TryHackMe | Full Kerberos chain: kerbrute → AS-REP Roasting → Kerberoasting → Pass-the-Ticket → Golden/Silver Ticket → Skeleton Key |
| [VulnNet: Roasted](https://tryhackme.com/room/vulnnetroasted) | TryHackMe | AS-REP Roasting + Kerberoasting in a real AD environment (impacket tools, hashcat) |
| [RazorBlack](https://tryhackme.com/room/raz0rblack) | TryHackMe | AS-REP Roasting, hash cracking, multiple Kerberos attack vectors |
| [AD Certificate Templates](https://tryhackme.com/room/adcertificatetemplates) | TryHackMe | ADCS misconfiguration (CVE-2022-26923) → certificate-based Domain Admin path |
| [Archetype](https://app.hackthebox.com/machines/Archetype) | HackTheBox | MSSQL xp_cmdshell → SMB credential recovery → WinRM → DCSync-style credential chain |

**→ PWK Notes:** [[PWK-Module-22-Attacking-AD-Authentication]] · [[AD-Authentication-Attack-Patterns]]

---

## Module 23 — Lateral Movement in Active Directory

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Archetype](https://app.hackthebox.com/machines/Archetype) | HackTheBox | Lateral movement using recovered credentials via SMB/WinRM; credential reuse chain |
| [Responder](https://app.hackthebox.com/machines/Responder) | HackTheBox | LFI → PHP include with UNC path → NTLM capture/relay → WinRM (mirrors relay chain in Module 22–24) |
| [Attacking Kerberos](https://tryhackme.com/room/attackingkerberos) | TryHackMe | Pass-the-Ticket and Golden Ticket lateral movement in practice |

**→ PWK Notes:** [[PWK-Module-23-Lateral-Movement-AD]] · [[AD-Lateral-Movement-Patterns]]

---

## Module 24 — Assembling the Pieces (Full Chain)

| Lab | Platform | Key Techniques |
|-----|----------|----------------|
| [Archetype](https://app.hackthebox.com/machines/Archetype) | HackTheBox | End-to-end Windows AD chain: MSSQL → SMB → WinRM → credential reuse → domain compromise |
| [Unified](https://app.hackthebox.com/machines/Unified) | HackTheBox | Log4Shell → MongoDB → admin panel RCE → UniFi credential extraction |
| [Wreath](https://tryhackme.com/room/wreath) | TryHackMe | Multi-machine network simulation: foothold → pivot → internal compromise (full chain) |

**→ PWK Notes:** [[PWK-Module-24-Assembling-the-Pieces]] · [[Pentest-Methodology-Mindset]]

---

## Other Relevant Labs (Cross-cutting)

| Lab | Platform | Relevant To |
|-----|----------|-------------|
| [Enumeration](https://tryhackme.com/room/enumerationpe) | TryHackMe | General host enumeration (Modules 8–21) |
| [Network Services](https://tryhackme.com/room/networkservices) | TryHackMe | SMB/FTP/Telnet/NFS exploitation basics |
| [Network Services 2](https://tryhackme.com/room/networkservices2) | TryHackMe | NFS/SMTP/MySQL exploitation |
| [CVE-2021-41773](https://tryhackme.com/room/cve202141773) | TryHackMe | Apache path traversal/RCE (Module 8/13) |
| [Atlassian CVE-2022-26134](https://tryhackme.com/room/confluence202226134) | TryHackMe | Confluence OGNL injection (similar to Module 19 BEYOND initial foothold) |
| [CVE-2022-26923](https://tryhackme.com/room/cve202226923) | TryHackMe | ADCS privilege escalation (Module 22 extension) |
| [Madeye's Castle](https://tryhackme.com/room/madeyescastle) | TryHackMe | SQLite injection → SSH privkey → sudo GTFOBins (Modules 10, 18) |

---

## HackTheBox Starting Point (Full Progression)

| Machine | Key Techniques | Module Relevance |
|---------|---------------|-----------------|
| [Meow](https://app.hackthebox.com/machines/Meow) | Telnet misconfiguration | Module 8 basics |
| [Fawn](https://app.hackthebox.com/machines/Fawn) | Anonymous FTP | Module 8 basics |
| [Dancing](https://app.hackthebox.com/machines/Dancing) | SMB anonymous access | Module 21 (SMB) |
| [Redeemer](https://app.hackthebox.com/machines/Redeemer) | Redis unauthenticated access | Module 9 |
| [Appointment](https://app.hackthebox.com/machines/Appointment) | SQL injection login bypass | Module 10 |
| [Sequel](https://app.hackthebox.com/machines/Sequel) | MySQL command execution | Module 10 |
| [Crocodile](https://app.hackthebox.com/machines/Crocodile) | FTP anonymous + web login | Module 8 |
| [Responder](https://app.hackthebox.com/machines/Responder) | LFI + NTLM relay + WinRM | Modules 19, 22, 23 |
| [Unified](https://app.hackthebox.com/machines/Unified) | Log4Shell + MongoDB | Modules 13, 24 |
| [Archetype](https://app.hackthebox.com/machines/Archetype) | MSSQL + SMB + WinRM + cred reuse | Modules 21, 22, 23, 24 |
| [Oopsie](https://app.hackthebox.com/machines/Oopsie) | IDOR + file upload + SUID | Modules 8, 18 |
| [Vaccine](https://app.hackthebox.com/machines/Vaccine) | SQLi + hash crack + sudo | Modules 10, 16, 18 |

---

## Connections

- [[PWK-Module-16-Password-Attacks]] · [[PWK-Module-17-Windows-PrivEsc]] · [[PWK-Module-18-Linux-PrivEsc]]
- [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] · [[PWK-Module-20-Metasploit-Framework]]
- [[PWK-Module-21-Active-Directory-Enumeration]] · [[PWK-Module-22-Attacking-AD-Authentication]]
- [[PWK-Module-23-Lateral-Movement-AD]] · [[PWK-Module-24-Assembling-the-Pieces]]
- [[Pentest-Methodology-Mindset]] · [[AD-Authentication-Attack-Patterns]] · [[AD-Lateral-Movement-Patterns]]
