---
title: "Password Cracking — 5-Step Methodology"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - password-attacks
  - hashcat
  - methodology
  - oscp
---

# Password Cracking — 5-Step Methodology

**Core idea:** Password cracking isn't random guessing. It's a systematic process: extract the hash, identify its type, build an intelligent wordlist based on what you know about the target, apply mutation rules to simulate real human password behaviour, then run the attack. Each step narrows the search space.

---

## The 5 Steps

```
1. EXTRACT   → get the hash out of the system
2. FORMAT    → remove filename prefixes (john tools add "filename:" prefix)
3. IDENTIFY  → determine hash algorithm → find hashcat mode number
4. PREPARE   → wordlist + rules (generic + target-specific)
5. ATTACK    → hashcat -m MODE hash.txt wordlist.txt -r rules.rule
```

---

## Step 1 — Extract: Where Hashes Live

| Source | Location / Method |
|--------|------------------|
| Linux shadow file | `/etc/shadow` (needs root) |
| Windows NTLM | Mimikatz, secretsdump.py, hashdump in Meterpreter |
| Web app database | MySQL dump → `password` column |
| KeePass database | `.kdbx` file → `keepass2john` |
| SSH private key | `id_rsa` file → `ssh2john` |
| Zip/7z archives | `zip2john`, `7z2john` |

---

## Step 2 — Format: The Prefix Problem

`john2hash` helper scripts (ssh2john, keepass2john, etc.) output hashes with a `filename:` prefix:
```
id_rsa:$sshng$6$16$7059e78a...
Database:$keepass$*2*60*0*d74e29...
```

Hashcat **cannot** process this prefix. Remove it:
```bash
sed -i 's/id_rsa://' ssh.hash
sed -i 's/Database://' keepass.hash
```

---

## Step 3 — Identify: Hash Type → Hashcat Mode

**Visual clues:**
- 32 hex → MD5 (mode 0)
- 40 hex → SHA1 (mode 100)
- 64 hex → SHA256 (mode 1400)
- `$2y$` / `$2a$` → bcrypt (mode 3200) — very slow
- `$6$` → SHA-512 crypt / Linux shadow (mode 1800)
- `$1$` → MD5 crypt (mode 500)
- Starts with `$keepass$` → mode 13400
- Starts with `$sshng$6$` → mode 22921

```bash
hashcat --help | grep -i "ntlm"    # → mode 1000
hashcat --help | grep -i "keepass" # → mode 13400
hash-identifier                    # interactive visual identifier
```

---

## Step 4 — Prepare: Smart Wordlists Beat Brute Force

**Generic starting point:** `rockyou.txt` (14M passwords)

**Target-specific intelligence** (if you found notes, emails, old passwords):
- Add known passwords/words to a custom list
- Analyse the password policy (length, special chars, numbers required)
- Build rules that match the policy

```bash
# Policy: capital + 3 numbers + special char
# Rule variants to cover common patterns:
c $1 $3 $7 $!    # → Umbrella137!
c $1 $3 $7 $@    # → Umbrella137@
c $1 $3 $7 $#    # → Umbrella137#
```

**Rule files by aggressiveness:**
1. `best64.rule` — fast first pass, high hit rate
2. `rockyou-30000.rule` — comprehensive
3. `dive.rule` / `d3ad0ne.rule` — last resort, slow

---

## Step 5 — Attack and Iterate

```bash
# First pass (fast)
hashcat -m MODE hash.txt rockyou.txt -r best64.rule --force

# Second pass (broader)
hashcat -m MODE hash.txt rockyou.txt -r rockyou-30000.rule --force

# Custom wordlist with custom rules
hashcat -m MODE hash.txt custom.txt -r custom.rule --force

# Restore interrupted session
hashcat --restore

# Show cracked passwords
hashcat -m MODE hash.txt --show
```

---

## The Human Password Pattern Insight

Users don't choose random passwords — they follow predictable patterns:
- Base word from personal life (name, hobby, company, sports team)
- Capitalise first letter
- Append numbers (often birth year, "1", "123", "2022")
- Append special character at the end (usually `!` or `@`)

This is why `c $1 c $! $1 c $@` rules combined with a personalised wordlist outperform brute force by orders of magnitude. **Know your target → build smarter rules.**

---

## Tool Selection: Hashcat vs John

| Situation | Use |
|-----------|-----|
| GPU available | Hashcat (150x+ faster) |
| No GPU / VM | John the Ripper |
| SSH key hash (token length errors in hashcat) | John the Ripper |
| Need format conversion scripts (ssh2john, keepass2john) | John suite provides them |

---

## Connections

- [[PWK-Module-16-Password-Attacks]] — full module with all Hydra + Hashcat + John commands
- [[PWK-Module-17-Windows-PrivEsc]] — NTLM hashes come from here; crack with mode 1000
- [[PWK-Module-18-Linux-PrivEsc]] — `/etc/shadow` SHA-512 hashes; crack with mode 1800
- Password reuse is a huge multiplier — cracked one hash → try same password on every other service

---

*From: PWK Module 16*
