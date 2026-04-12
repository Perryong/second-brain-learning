---
title: "Web Directory & File Enumeration — Tools and Methodology"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - enumeration
  - web
  - gobuster
  - feroxbuster
  - methodology
  - oscp
related:
  - "[[Nmap-Enumeration-Methodology]]"
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
  - "[[File-Upload-Filter-Bypass-Techniques]]"
---

# Web Directory & File Enumeration — Tools and Methodology

**Core idea:** Every web target hides paths that don't appear in the visible UI. Directory enumeration is never optional — it's how you find the admin panel, the hidden upload endpoint, the `.php` webshell, the `/secret/` folder, and the `robots.txt` disavowals. Run it the moment you confirm a web server is alive.

---

## Core Tool Comparison

| Tool | Best For | Notes |
|------|----------|-------|
| `gobuster` | Clean, fast directory bruteforce | Most commonly seen in PG writeups |
| `feroxbuster` | Recursive enumeration + fast | Auto-recurses into discovered dirs |
| `dirb` / `dirbuster` | Classic fallback | Slower, but works everywhere |
| `wfuzz` | Parameter fuzzing, header fuzzing | Flexible but verbose syntax |
| `ffuf` | Fast, flexible, scriptable | Good for param/vhost fuzzing too |
| `nikto` | Technology fingerprinting + common vulns | Noisy but useful for quick overview |

---

## Command Reference

### Gobuster (Directory)
```bash
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -o gobuster.txt

# With authentication
gobuster dir -u http://TARGET -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -U admin -P password

# HTTPS (ignore TLS errors)
gobuster dir -u https://TARGET -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -k
```

### Feroxbuster (Recursive)
```bash
feroxbuster -u http://TARGET -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt

# Quiet mode (useful for scripting)
feroxbuster -u http://TARGET -q --no-state
```

### WFUZZ (Parameter Fuzzing)
```bash
# Fuzz GET parameters for LFI
wfuzz -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  --hl 0 "http://TARGET/page.php?FUZZ=test"

# Fuzz parameter values (LFI payloads)
wfuzz -c -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
  "http://TARGET/page.php?param=FUZZ"
```

---

## Key Patterns from Proving Grounds

### 1. Always Check robots.txt First (Manual Step)
```bash
curl http://TARGET/robots.txt
```
Several PG boxes hid critical paths in robots.txt:
- **DriftingBlues6:** `Disallow: /textpattern/textpattern` + hint about `.zip` extension fuzzing
- **Stapler:** `Disallow: /admin112233/` and `Disallow: /blogblog/` (WordPress)
- **Inclusiveness:** robots.txt revealed `/secret_information/` BUT was only accessible to GoogleBot user-agent

### 2. User-Agent Bypass (Inclusiveness Pattern)
```bash
# When robots.txt blocks scrapers but not human browsers
curl -s -A "Googlebot/2.1 (+http://www.google.com/bot.html)" http://TARGET/robots.txt
```

### 3. Check Page Source for CMS Version
```html
<!-- Common in PG boxes — reveals exact vulnerable version -->
<meta name="Generator" content="Drupal 7" />
<meta name="generator" content="Koken 0.22.24" />
```

### 4. Directory Fuzzing Reveals Hidden Uploads/Admin
From PG boxes, common high-value paths found:
- `/uploads/` — direct file upload → RCE (Access box)
- `/admin/` / `/wp-admin/` — login panel
- `/secret/` — hidden PHP endpoints (EvilBox-One)
- `/console/` — admin tools (Ha-natraj had LFI-vulnerable `file.php` here)
- `/textpattern/textpattern/` — CMS admin (DriftingBlues6)
- `/api/user/` — exposed credentials in JSON (Hunit)

### 5. File Extension Fuzzing Matters
```bash
# Don't just look for directories — PHP files, backups, configs
gobuster dir -u http://TARGET -w wordlist.txt -x php,txt,bak,zip,conf,config

# Found in PG boxes:
# evil.php (EvilBox-One) — LFI endpoint
# wp-config.php (Stapler) — extracted via WordPress LFI exploit
# config.py, syncer.py (Djinn3) — cron job scripts
```

---

## Wordlist Quick Reference

| Path | Use |
|------|-----|
| `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` | General directories |
| `/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt` | Comprehensive dirs |
| `/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files.txt` | Files with extensions |
| `/usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt` | GET/POST param names |
| `/usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt` | LFI payloads |

---

## WPScan — WordPress-Specific Enumeration

```bash
# Enumerate plugins, themes, and users
wpscan --url https://TARGET/wordpress/ --enumerate p,t,u

# Brute-force credentials
wpscan --url https://TARGET/wordpress/ -P /usr/share/wordlists/rockyou.txt

# Ignore SSL errors
wpscan --url https://TARGET/wordpress/ --disable-tls-checks --enumerate p,t,u
```

**From Stapler writeup:** WPScan found the vulnerable `advanced-video-embed` plugin and enumerated valid usernames. Even non-admin accounts expose the plugin inventory.

---

## Connections

- [[LFI-to-RCE-Log-Poisoning-Pattern]] — finding `/console/file.php` leads directly here
- [[File-Upload-Filter-Bypass-Techniques]] — once `/uploads/` is discovered
- [[SSTI-Jinja2-Exploitation]] — Werkzeug/Jinja2 endpoints found via directory enumeration
- [[SMB-Enumeration-Exploitation]] — credentials found in SMB often unlock web panels
- [[Nmap-Enumeration-Methodology]] — always precedes this step

---

*Extracted from 70+ Proving Grounds writeups (PG Play + PG Practice, 2023–2024)*
