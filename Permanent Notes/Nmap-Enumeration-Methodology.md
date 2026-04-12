---
title: "Nmap — Enumeration Methodology and Command Reference"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - enumeration
  - nmap
  - methodology
  - oscp
related:
  - "[[Web-Directory-Enumeration]]"
  - "[[SMB-Enumeration-Exploitation]]"
  - "[[Pentest-Methodology-Mindset]]"
---

# Nmap — Enumeration Methodology and Command Reference

**Core idea:** Nmap is always the first step. The goal of the initial scan is speed — find all open ports before doing deep service identification. A two-phase approach (fast full-port scan → targeted service scan) consistently outperforms trying to do everything in one pass.

---

## The Two-Phase Approach

```bash
# Phase 1: Fast full-port scan (find open ports)
nmap -p- --open -sT -v -oN nmap-ports TARGET

# Phase 2: Deep service scan on discovered ports only
nmap -sV -sC -p 22,80,443,8080 -oN nmap-services TARGET
```

**Why two phases:** `-p-` on all 65535 ports with `-sV` is extremely slow. Find the ports first, then do slow/noisy service detection only on the ports you know are open.

---

## Core Flags Reference

| Flag | Meaning | When to Use |
|------|---------|-------------|
| `-p-` | Scan all 65535 ports | Initial discovery — always |
| `--open` | Show only open ports | Reduces noise |
| `-sT` | TCP Connect scan (full 3-way handshake) | When SYN scan is blocked or you need accuracy |
| `-sS` | SYN Stealth scan | Faster, but needs root |
| `-sV` | Service version detection | Phase 2 — identifies exact software versions |
| `-sC` | Default NSE scripts | Runs safe enumeration scripts (banners, auth, etc.) |
| `-A` | Aggressive (`-sV -sC -O --traceroute`) | Comprehensive but noisy |
| `-v` | Verbose output | Shows results as they come in |
| `-oN filename` | Normal output to file | Always save your scans |
| `-oG filename` | Grepable output | Useful for scripting |

---

## Service-Specific Discoveries from PG Writeups

### FTP (Port 21) — Key Scripts
```bash
nmap -sC -sV -p 21 TARGET
# Look for: ftp-anon: Anonymous FTP login allowed
# If anonymous login is allowed — this is your first foothold
```

### SMB (Ports 139/445) — Key Scripts
```bash
nmap -sC -sV -p 139,445 TARGET
# nmap automatically runs: smb-os-discovery, smb2-security-mode
# Look for: signing:disabled (relay attack possible)
```

### SMTP (Port 25)
```bash
nmap -sC -sV -p 25 TARGET
# smtp-commands shows: VRFY (user enumeration), EXPN (list aliases)
# Look for: specific software versions (Sendmail, Postfix, OpenSMTPD)
```

### SSH (Port 22 / Non-standard)
```bash
# SSH is often moved to non-standard ports in CTFs
# Common alternates seen: 61000, 43022, 60000
# Always scan for these — they show up in full-port scans only
```

---

## Important Takeaways from Proving Grounds

From 70+ PG writeups, these patterns appear repeatedly:

1. **Non-standard ports are goldmines.** SSH on 61000, HTTP on 8080/8000/9998, custom services on 31337/43500. A scan limited to top 1000 ports misses all of these.

2. **Multiple HTTP servers = multiple attack surfaces.** Boxes like Stapler (ports 80 and 12380), Djinn3 (ports 80, 5000, 31337), and Algernon (ports 80 and 9998) each had the actual vulnerability on a non-standard web port.

3. **Service version = searchsploit input.** Always record the exact version strings. `Koken 0.22.24`, `SaltStack 3000.1`, `Nagios XI 5.6.0` — paste these directly into searchsploit or Google.

4. **Anonymous FTP with write access is rare but game-changing.** If FTP allows anonymous login AND anonymous write to a web-accessible directory, that's an LFI → RCE chain in one step.

---

## SNMP Enumeration

```bash
# SNMP runs on UDP 161 — Nmap TCP scans miss it entirely
nmap -sU -p 161 TARGET
snmp-check TARGET                         # full SNMP enumeration
snmpwalk -c public -v1 TARGET             # manual walk
```

**From ClamAV writeup:** SNMP revealed `clamav-milter` running with `black-hole-mode` enabled — the exact vulnerable configuration — before any web interaction at all.

---

## Connections

- [[Web-Directory-Enumeration]] — next step after finding open HTTP ports
- [[SMB-Enumeration-Exploitation]] — SMB enumeration after finding ports 139/445
- [[FTP-Exploitation-Techniques]] — FTP attack patterns after finding port 21
- [[Exploit-Discovery-Workflow]] — how to use version strings to find exploits
- [[Pentest-Methodology-Mindset]] — the bigger picture of why enumeration matters

---

*Extracted from 70+ Proving Grounds writeups (PG Play + PG Practice, 2023–2024)*
