---
title: "Proving Grounds — Master Index & Technique Map"
created: "2026-04-11"
updated: "2026-04-11"
type: reference-note
status: active
tags:
  - reference
  - proving-grounds
  - oscp
  - index
  - ctf
related:
  - "[[OSCP-Certification-Sprint]]"
  - "[[Nmap-Enumeration-Methodology]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
---

# Proving Grounds — Master Index & Technique Map

This note maps all 70+ PG Practice/Play machines to the techniques used. Use this as a **quick technique lookup**: if you're stuck on a box using technique X, find another box that used X and review its pattern.

---

## Technique → Machine Reference

### Enumeration
| Technique | Machine(s) |
|-----------|-----------|
| Anonymous FTP + hidden directory | OnSystemShellDredd |
| Non-standard SSH port (61000, 43022) | OnSystemShellDredd, Hunit |
| SNMP revealing vulnerable service | ClamAV |
| Multiple HTTP services on different ports | Djinn3, Algernon, Stapler, Monitoring |
| WordPress enumeration (WPScan) | Stapler, Blogger, DC-2 |
| robots.txt bypass (User-Agent) | Inclusiveness |
| API endpoint exposing credentials | Hunit |
| Source code version fingerprinting | DC-1 (Drupal), Photographer (Koken) |

### Web Exploitation
| Technique | Machine(s) |
|-----------|-----------|
| LFI → /etc/passwd | EvilBox-One, Ha-natraj, Inclusiveness |
| LFI → SSH key theft | EvilBox-One |
| LFI + SSH Log Poisoning → RCE | Ha-natraj |
| SSTI (Jinja2/Werkzeug) | Djinn3 |
| File Upload + .htaccess bypass | Access |
| File Upload + Burp intercept | Photographer |
| WordPress plugin file upload | Stapler |
| SQL Injection → MySQL → shell | Banzai, Stapler |
| Grafana directory traversal (CVE-2021-43798) | Fanatastic |
| SaltStack RCE (CVE-2020-11651) | Twiggy |
| Drupageddon (Metasploit) | DC-1 |
| Koken CMS file upload | Photographer |
| Nagios XI RCE (Metasploit) | Monitoring |
| Shellshock | Sumo |
| XPath Injection | Voter1 |

### Initial Foothold
| Technique | Machine(s) |
|-----------|-----------|
| SSH key stolen from FTP | OnSystemShellDredd |
| Credentials in SMB share | Photographer, Dawn |
| Credential in zip file (fcrackzip) | DriftingBlues6 |
| Credentials from API JSON response | Hunit |
| Default credentials (FTP brute) | Banzai |
| PHP reverse shell via upload | Photographer, Access, Stapler |
| Python reverse shell via SSTI | Djinn3 |
| Netcat reverse shell via SMB cron | Dawn |
| RCE via Perl exploit (ClamAV) | ClamAV |
| Writable /etc/passwd via SaltStack | Twiggy |

### Linux Privilege Escalation
| Technique | Machine(s) |
|-----------|-----------|
| SUID: find | DC-1 |
| SUID: php | Photographer |
| SUID: mawk | OnSystemShellDredd |
| SUID: cpulimit | OnSystemShellDredd |
| SUID: zsh | Dawn |
| sudo: nmap → .nse script | Ha-natraj |
| sudo: apt-get → pager escape | Djinn3 |
| Cron: writable script | Stapler, Dawn |
| Cron: writable SMB share | Dawn |
| Cron: git push injection | Hunit |
| Cron: JSON config manipulation | Djinn3 |
| Writable /etc/passwd | EvilBox-One, Twiggy |
| Dirty COW (CVE-2016-5195) | DriftingBlues6 |
| PATH hijacking | Inclusiveness |
| Disk group → debugfs | Fanatastic |
| MySQL UDF → sys_exec | Banzai |
| Apache config user change | Ha-natraj |
| Fail2ban action script | SunsetMidnight (implied) |
| Kernel exploit | Sumo, Cybersploit1 |

### Windows / Active Directory
| Technique | Machine(s) |
|-----------|-----------|
| Kerberoasting (Rubeus) | Access |
| SeManageVolumePrivilege → DLL hijack | Access |
| NTLM Hash extraction | Access |
| Invoke-RunasCs lateral movement | Access |
| WinRM (Evil-WinRM) | Hetemit, others |

### Password Attacks
| Technique | Machine(s) |
|-----------|-----------|
| SSH key cracking (ssh2john) | EvilBox-One |
| NTLM hash cracking (john/hashcat) | Access |
| Zip file cracking (fcrackzip) | DriftingBlues6 |
| MySQL hash extraction | Stapler |
| FTP credential brute force (hydra) | Banzai |
| SSH credential brute force (ncrack) | Hunit |

---

## Machines by Difficulty Category

### Easy / Good for Fundamentals
- OnSystemShellDredd — FTP anon, SUID mawk/cpulimit
- DC-1 — Drupageddon Metasploit, SUID find
- Photographer — SMB creds, Koken upload, SUID php
- EvilBox-One — LFI, ssh key, writable /etc/passwd
- Cybersploit1 — base64 decode, kernel exploit

### Medium / Chaining Required
- Access — AD, Kerberoast, DLL hijack (Windows AD)
- Ha-natraj — LFI + SSH log poisoning, Apache conf
- Dawn — SMB + cron combo
- DriftingBlues6 — zip cracking + Dirty COW
- Inclusiveness — User-Agent bypass + PATH hijacking
- Djinn3 — SSTI + cron JSON injection
- Twiggy — SaltStack RCE

### Hard / Advanced Chains
- Banzai — multi-service, MySQL UDF
- Hunit — API creds + SMB + git cron injection
- Fanatastic — Grafana CVE + disk group debugfs
- Stapler — WordPress + MySQL + cron
- Algernon — SolarWinds Serv-U exploit

---

## Methodology Notes Per Machine Category

### "Just Run LinPEAS" is Not a Strategy
The boxes that get people stuck are ones where `sudo -l` shows nothing and linpeas output is long. The actual path is usually one of:
- A writable cron script visible in `ls /etc/cron.*`
- A SUID binary that's uncommon (zsh, mawk, cpulimit)
- A service running as root with a writable config file
- A group membership (disk, docker, lxd) that provides filesystem access

### The "Two Tab" Rule
Always have GTFObins (https://gtfobins.github.io/) open in one tab and HackTricks (https://book.hacktricks.xyz/) in another. The moment you find a binary in SUID or sudo, check GTFObins immediately.

---

## Key Notes for This Box Set

- All machines here are **Proving Grounds Play** or **Proving Grounds Practice**
- PG Practice machines are closer to OSCP exam difficulty
- The author (thevillagehacker) consistently uses the two-phase Nmap approach
- Every box follows: Enumerate → Foothold → PrivEsc
- **Most boxes required NO Metasploit** — good OSCP manual practice

---

## Technique Notes (Linked)

### Enumeration Notes
- [[Nmap-Enumeration-Methodology]]
- [[Web-Directory-Enumeration]]
- [[SMB-Enumeration-Exploitation]]
- [[FTP-Exploitation-Techniques]]

### Exploitation Notes
- [[File-Upload-to-RCE-Patterns]]
- [[File-Upload-Filter-Bypass-Techniques]]
- [[LFI-to-RCE-Log-Poisoning-Pattern]]
- [[SSTI-Jinja2-Exploitation]]
- [[SQLi-to-RCE-Attack-Chains]]
- [[Searchsploit-and-CVE-Workflow]]

### Privilege Escalation Notes
- [[Linux-PrivEsc-Decision-Tree]]
- [[Advanced-Linux-PrivEsc-Patterns]]
- [[GTFObins-SUID-Quick-Reference]]
- [[Kernel-Exploit-Methodology]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Kerberoasting-Attack-Chain]]
- [[AD-Authentication-Attack-Patterns]]

### Shells & Tools
- [[Reverse-Shell-Reference]]
- [[Password-Cracking-Methodology]]
- [[Metasploit-Mental-Model]]
- [[Exploit-Discovery-Workflow]]

---

*Covers 70+ machines: PG Play (2023-07 through 2023-12) + PG Practice (2023-08 through 2024-06)*
*All writeups by thevillagehacker (@thevillagehackr)*
