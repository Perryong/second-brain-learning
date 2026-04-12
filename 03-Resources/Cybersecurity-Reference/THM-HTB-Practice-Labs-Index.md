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

**Purpose:** Map every completed lab to the relevant PWK module. Walkthroughs live in `03-Resources/Practice-Walkthroughs/`. Click any link to open the full walkthrough.

---

## Module 8–10 — Web App Attacks / SQL Injection

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Game Zone]] | TryHackMe | Manual SQLi → sqlmap → hash dump → hashcat → SSH reverse tunnel |
| [[Bookstore]] | TryHackMe | REST API enumeration, fuzzing hidden parameters |
| [[NoSQL injection Basics]] | TryHackMe | MongoDB NoSQL injection bypass |
| [[Road]] | TryHackMe | IDOR → file upload → SUID binary PrivEsc |
| [[Authentication Bypass]] | TryHackMe | Username enumeration, brute-force, logic flaws |
| [[Command Injection]] | TryHackMe | OS command injection via web form |
| [[Upload Vulnerabilities]] | TryHackMe | File upload bypass (MIME, extension, magic bytes) |
| [[Unified]] | HackTheBox | Log4Shell (CVE-2021-44228) → MongoDB cred extraction |
| [[Sequel]] | HackTheBox | MySQL unauthenticated access → SQL commands |
| [[Appointment]] | HackTheBox | SQL injection login bypass |

**→ PWK Notes:** [[PWK-Module-08-Web-App-Attacks]] · [[PWK-Module-09-Common-Web-App-Attacks]] · [[PWK-Module-10-SQL-Injection]]

---

## Module 11–12 — Phishing / Client-Side Attacks

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Advent of Cyber 2022]] | TryHackMe | Phishing analysis, macro-based attacks (Task 3–4) |
| [[Brute Force Heroes]] | TryHackMe | Credential phishing simulation |

**→ PWK Notes:** [[PWK-Module-11-Phishing-Basics]] · [[PWK-Module-12-Client-Side-Attacks]]

---

## Module 13–14 — Public Exploits / Buffer Overflows

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Buffer Overflow Prep]] | TryHackMe | 10× OVERFLOW exercises — complete x86 Windows BoF methodology (EIP, bad chars, JMP ESP, shellcode) |
| [[Buffer Overflows]] | TryHackMe | Basic stack overflow introduction |
| [[Brainstorm]] | TryHackMe | x86 Windows stack buffer overflow — full exploit development |
| [[Brainpan 1]] | TryHackMe | Linux/Windows BoF with OSCP-style challenge format |
| [[Exploit Vulnerabilities]] | TryHackMe | Finding and using public exploits (searchsploit workflow) |

**→ PWK Notes:** [[PWK-Module-13-Locating-Public-Exploits]] · [[PWK-Module-14-Fixing-Exploits]]

---

## Module 16 — Password Attacks

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Password Attacks]] | TryHackMe | CUPP/crunch/cewl wordlist generation, Hydra/Medusa, hashcat modes |
| [[Game Zone]] | TryHackMe | sqlmap hash extraction → hashcat cracking |
| [[Capture!]] | TryHackMe | Rate-limited brute-force with CAPTCHA bypass |
| [[Brute]] | TryHackMe | HTTP brute-force and credential stuffing |
| [[Vaccine]] | HackTheBox | Hash cracking (MD5 → plaintext) → sudo escalation |

**→ PWK Notes:** [[PWK-Module-16-Password-Attacks]]

---

## Module 17 — Windows Privilege Escalation

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Windows Privilege Escalation]] | TryHackMe | Service misconfigs, AlwaysInstallElevated, scheduled tasks, credential harvesting |
| [[Steel Mountain]] | TryHackMe | Rejetto HFS CVE-2014-6287 → unquoted service path / insecure binary PrivEsc |
| [[Alfred]] | TryHackMe | Jenkins exploit → Meterpreter → token impersonation (incognito) → SYSTEM |
| [[Internal]] | TryHackMe | WordPress → Jenkins → SSH privkey → root (full internal pentest) |
| [[Abusing Windows Internals]] | TryHackMe | DLL injection, process hollowing, token manipulation |
| [[Windows Local Persistence]] | TryHackMe | Registry autorun, backdoor service, RDP backdoor techniques |
| [[Oopsie]] | HackTheBox | IDOR → file upload → SUID binary PrivEsc |

**→ PWK Notes:** [[PWK-Module-17-Windows-PrivEsc]]

---

## Module 18 — Linux Privilege Escalation

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Common Linux Privesc]] | TryHackMe | SUID, sudo misconfigs, writable /etc/passwd, cron jobs, path hijacking |
| [[Linux Local Enumeration]] | TryHackMe | Systematic local enumeration methodology |
| [[0day]] | TryHackMe | Shellshock (CVE-2014-6271) → Linux kernel PrivEsc (overlayfs) |
| [[Internal]] | TryHackMe | SSH private key discovery after pivoting through internal services |
| [[Oh My WebServer]] | TryHackMe | CVE-2021-41773 Apache RCE → Docker escape → root |
| [[Vaccine]] | HackTheBox | sudo misconfiguration escalation |

**→ PWK Notes:** [[PWK-Module-18-Linux-PrivEsc]]

---

## Module 19 — Port Redirection and SSH Tunneling

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Wreath]] | TryHackMe | Full multi-hop pivoting: SSH -L/-R, Socat, Chisel, sshuttle, plink.exe across 3-machine network |
| [[Extending Your Network]] | TryHackMe | Port forwarding and firewall fundamentals |
| [[Game Zone]] | TryHackMe | SSH reverse tunnel (-R) to expose internal Webmin to Kali |
| [[Badbyte]] | TryHackMe | SSH tunneling to reach services on a segmented internal network |

**→ PWK Notes:** [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] · [[Pivoting-Mental-Model]]

---

## Module 20 — The Metasploit Framework

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Metasploit]] | TryHackMe | msfdb init, msfconsole basics, auxiliary modules, MS17-010 (EternalBlue) |
| [[Metasploit Exploitation]] | TryHackMe | db_nmap integration, vulnerability scanning, msfvenom payload generation |
| [[Metasploit  Meterpreter]] | TryHackMe | Meterpreter in-memory architecture, process migration, channel commands |
| [[Alfred]] | TryHackMe | Jenkins → Meterpreter → getsystem via token impersonation |
| [[AV Evasion Shellcode]] | TryHackMe | msfvenom shellcode encoding/packing to evade AV detection |

**→ PWK Notes:** [[PWK-Module-20-Metasploit-Framework]] · [[Metasploit-Mental-Model]]

---

## Module 21 — Active Directory Enumeration

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Active Directory Basics]] | TryHackMe | Domain structure, users/groups/OUs, GPOs, trusts |
| [[Attacking Kerberos]] | TryHackMe | Kerbrute user enumeration, SPN discovery, BloodHound context |
| [[VulnNet Roasted]] | TryHackMe | Null session SMB → share enumeration → user discovery → credential attacks |
| [[RazorBlack]] | TryHackMe | AD enumeration pipeline leading into authentication attacks |

**→ PWK Notes:** [[PWK-Module-21-Active-Directory-Enumeration]] · [[AD-Enumeration-to-Attack-Path]]

---

## Module 22 — Attacking Active Directory Authentication

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Attacking Kerberos]] | TryHackMe | Full Kerberos chain: AS-REP Roasting → Kerberoasting → Pass-the-Ticket → Golden/Silver Ticket → Skeleton Key |
| [[VulnNet Roasted]] | TryHackMe | AS-REP Roasting + Kerberoasting in real AD environment (impacket tools, hashcat) |
| [[RazorBlack]] | TryHackMe | AS-REP Roasting, hash cracking, multiple Kerberos attack vectors |
| [[AD Certificate Templates]] | TryHackMe | ADCS misconfiguration (CVE-2022-26923) → certificate-based Domain Admin path |
| [[Archetype]] | HackTheBox | MSSQL xp_cmdshell → SMB credential recovery → WinRM → DCSync-style credential chain |

**→ PWK Notes:** [[PWK-Module-22-Attacking-AD-Authentication]] · [[AD-Authentication-Attack-Patterns]]

---

## Module 23 — Lateral Movement in Active Directory

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Archetype]] | HackTheBox | Lateral movement using recovered credentials via SMB/WinRM; credential reuse chain |
| [[Responder]] | HackTheBox | LFI → PHP include with UNC path → NTLM capture → WinRM shell |
| [[Attacking Kerberos]] | TryHackMe | Pass-the-Ticket and Golden Ticket lateral movement in practice |

**→ PWK Notes:** [[PWK-Module-23-Lateral-Movement-AD]] · [[AD-Lateral-Movement-Patterns]]

---

## Module 24 — Assembling the Pieces (Full Chain)

| Walkthrough | Platform | Key Techniques |
|-------------|----------|----------------|
| [[Archetype]] | HackTheBox | End-to-end Windows AD chain: MSSQL → SMB → WinRM → credential reuse → domain compromise |
| [[Unified]] | HackTheBox | Log4Shell → MongoDB → admin panel RCE → UniFi credential extraction |
| [[Wreath]] | TryHackMe | Multi-machine network simulation: foothold → pivot → internal compromise (full chain) |

**→ PWK Notes:** [[PWK-Module-24-Assembling-the-Pieces]] · [[Pentest-Methodology-Mindset]]

---

## HackTheBox Starting Point — Full Progression

| Walkthrough | Key Techniques | Module Relevance |
|-------------|---------------|-----------------|
| [[Meow]] | Telnet misconfiguration | Module 8 basics |
| [[Fawn]] | Anonymous FTP | Module 8 basics |
| [[Dancing]] | SMB anonymous access | Module 21 (SMB) |
| [[Redeemer]] | Redis unauthenticated access | Module 9 |
| [[Appointment]] | SQL injection login bypass | Module 10 |
| [[Sequel]] | MySQL command execution | Module 10 |
| [[Crocodile]] | FTP anonymous + web login | Module 8 |
| [[Responder]] | LFI + NTLM relay + WinRM | Modules 19, 22, 23 |
| [[Unified]] | Log4Shell + MongoDB | Modules 13, 24 |
| [[Archetype]] | MSSQL + SMB + WinRM + cred reuse | Modules 21, 22, 23, 24 |
| [[Oopsie]] | IDOR + file upload + SUID | Modules 8, 18 |
| [[Vaccine]] | SQLi + hash crack + sudo | Modules 10, 16, 18 |

---

## Other Relevant Labs (Cross-cutting)

| Walkthrough | Platform | Relevant To |
|-------------|----------|-------------|
| [[Enumeration]] | TryHackMe | General host enumeration (Modules 8–21) |
| [[Network Services]] | TryHackMe | SMB/FTP/Telnet/NFS exploitation basics |
| [[Network Services 2]] | TryHackMe | NFS/SMTP/MySQL exploitation |
| [[CVE-2021-41773]] | TryHackMe | Apache path traversal/RCE (Module 8/13) |
| [[Atlassian, CVE-2022-26134]] | TryHackMe | Confluence OGNL injection (similar to Module 19 BEYOND initial foothold) |
| [[CVE-2022-26923]] | TryHackMe | ADCS privilege escalation (Module 22 extension) |
| [[Madeye's Castle]] | TryHackMe | SQLite injection → SSH privkey → sudo GTFOBins (Modules 10, 18) |

---

## Connections

- [[PWK-Module-16-Password-Attacks]] · [[PWK-Module-17-Windows-PrivEsc]] · [[PWK-Module-18-Linux-PrivEsc]]
- [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] · [[PWK-Module-20-Metasploit-Framework]]
- [[PWK-Module-21-Active-Directory-Enumeration]] · [[PWK-Module-22-Attacking-AD-Authentication]]
- [[PWK-Module-23-Lateral-Movement-AD]] · [[PWK-Module-24-Assembling-the-Pieces]]
- [[Pentest-Methodology-Mindset]] · [[AD-Authentication-Attack-Patterns]] · [[AD-Lateral-Movement-Patterns]]
