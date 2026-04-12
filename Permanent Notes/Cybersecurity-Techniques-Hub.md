---
title: "Cybersecurity Techniques Hub — All Permanent Notes Index"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - index
  - hub
  - oscp
  - methodology
related:
  - "[[OSCP-Certification-Sprint]]"
  - "[[Pentest-Methodology-Mindset]]"
  - "[[Proving-Grounds-Master-Index]]"
---

# Cybersecurity Techniques Hub — All Permanent Notes Index

This is the master hub for all cybersecurity technique notes in this vault. Use it as a starting point when you know what technique you're looking for but not which note it's in.

---

## Phase 1: Reconnaissance & Enumeration

- **[[Nmap-Enumeration-Methodology]]** — Two-phase scan approach, flag reference, service-specific scripts, key patterns from 70+ PG boxes
- **[[Web-Directory-Enumeration]]** — gobuster/feroxbuster/ffuf/wfuzz, robots.txt bypass, WordPress WPScan, key paths to find
- **[[SMB-Enumeration-Exploitation]]** — smbclient, enum4linux, writable shares, credential files, cron via SMB
- **[[FTP-Exploitation-Techniques]]** — anonymous login, hidden directories, writable → web path → RCE, hydra brute force

---

## Phase 2: Exploitation / Initial Foothold

- **[[File-Upload-to-RCE-Patterns]]** — .htaccess bypass, Burp intercept rename, CMS upload (WordPress, Koken), ASPX shells
- **[[File-Upload-Filter-Bypass-Techniques]]** — double extension, MIME spoofing, magic bytes, null bytes (all evasion methods)
- **[[LFI-to-RCE-Log-Poisoning-Pattern]]** — log files to poison, 4-command attack chain, difference from directory traversal
- **[[SSTI-Jinja2-Exploitation]]** — detection payloads, Jinja2 RCE, full Djinn3 attack chain, other template engines
- **[[SQLi-to-RCE-Attack-Chains]]** — SQL injection to file write to webshell, UNION-based, error-based
- **[[Searchsploit-and-CVE-Workflow]]** — version-to-exploit pipeline, 10+ CVEs from PG boxes, adapting exploits
- **[[Exploit-Discovery-Workflow]]** — searchsploit, Google, GitHub PoC, exploit adaptation

---

## Phase 3: Privilege Escalation

### Linux PrivEsc
- **[[Linux-PrivEsc-Decision-Tree]]** — Priority order (sudo → SUID → cron → /etc/passwd → kernel), command reference, mental models
- **[[GTFObins-SUID-Quick-Reference]]** — Binaries seen in PG: mawk, cpulimit, find, php, zsh, nmap, apt-get, and more
- **[[Advanced-Linux-PrivEsc-Patterns]]** — PATH hijacking, disk group/debugfs, MySQL UDF, Apache config abuse, git cron injection
- **[[Kernel-Exploit-Methodology]]** — Dirty COW (CVE-2016-5195), Pwnkit, version detection, compile on target

### Windows PrivEsc
- **[[Windows-PrivEsc-Decision-Tree]]** — SeImpersonatePrivilege, unquoted service paths, AlwaysInstallElevated, registry
- **[[Kerberoasting-Attack-Chain]]** — SPN detection, Rubeus, Impacket, crack mode 13100, SeManageVolumePrivilege

### Active Directory
- **[[AD-Authentication-Attack-Patterns]]** — Pass-the-Hash, Pass-the-Ticket, NTLM relay, credential dumping
- **[[AD-Enumeration-to-Attack-Path]]** — BloodHound, PowerView, attack path identification
- **[[AD-Lateral-Movement-Patterns]]** — WMI, PsExec, WinRM, Evil-WinRM, CrackMapExec

---

## Tools & Supporting Techniques

- **[[Reverse-Shell-Reference]]** — bash, python, PHP, netcat, msfvenom, full TTY upgrade, file transfer
- **[[Password-Cracking-Methodology]]** — 5-step methodology, hash identification, hashcat modes, wordlists
- **[[Metasploit-Mental-Model]]** — module usage, handler setup, staged vs. stageless
- **[[Pivoting-Mental-Model]]** — chisel, socat, SSH tunneling, proxychains

---

## Methodology Notes

- **[[Pentest-Methodology-Mindset]]** — The assembling mindset, cyclical enumeration, never mark a target "done"
- **[[API-Testing-Methodology]]** — API endpoint enumeration, testing patterns
- **[[Credential-Phishing-Clone-Methodology]]** — credential harvesting techniques

---

## Practice Resources

- **[[Proving-Grounds-Master-Index]]** — 70+ PG machines mapped to techniques (in 03-Resources)
- **[[OSCP-Certification-Sprint]]** — active project tracking exam prep

---

## How to Use This Hub

**When you're on a box and stuck:**
1. Check what phase you're in (recon / foothold / privesc)
2. Look at the relevant section above
3. Open the note → find the pattern → apply it

**When you learn a new technique:**
1. Find the most relevant existing note
2. Add your findings to the "Why It Matters" or "Examples" section
3. Update the PG Master Index with the box name and technique

**The most important principle:** No orphan notes. Every new technique note should link to at least 3 others.

---

*Created 2026-04-11 from extraction of 70+ Proving Grounds writeups (PG Play + PG Practice, 2023-07 to 2024-06)*
