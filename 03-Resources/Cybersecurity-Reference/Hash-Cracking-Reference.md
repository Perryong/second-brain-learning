---
title: "Hash Cracking Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - hash-cracking
  - hashcat
  - john
  - password-attacks
---

# Hash Cracking Reference

Reference for hash cracking with hashcat and John the Ripper, including hash types, attack modes, wordlists, and rules. See also: [[AD-Credential-Attacks]], [[AD-Kerberos-Attacks]], [[Mimikatz-Reference]].

---

## Table of Contents

- [[#Hash Identification]]
- [[#Hashcat]]
- [[#John the Ripper]]
- [[#Hash Type Reference]]
- [[#Attack Modes]]
- [[#Wordlists and Rules]]
- [[#Rainbow Tables]]
- [[#Online Resources]]
- [[#Tips and Tricks]]

---

## Hash Identification

```bash
# hashid
hashid 'HASH_STRING'
hashid -m 'HASH_STRING'  # show hashcat mode

# hash-identifier
hash-identifier HASH_STRING

# haiti (most accurate)
haiti HASH_STRING
```

---

## Hashcat

### Installation

```bash
# Kali/Ubuntu
apt install hashcat

# Download (latest with GPU support)
wget https://hashcat.net/files/hashcat-6.x.x.7z

# Verify install
hashcat --version
hashcat -I   # List OpenCL/CUDA devices
```

### Basic Usage

```bash
# Dictionary attack
hashcat -m <MODE> hashes.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m <MODE> hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Mask attack (brute force pattern)
hashcat -m <MODE> hashes.txt -a 3 ?u?l?l?l?d?d?d?d

# Combinator attack
hashcat -m <MODE> hashes.txt -a 1 wordlist1.txt wordlist2.txt

# Hybrid (wordlist + mask)
hashcat -m <MODE> hashes.txt -a 6 wordlist.txt ?d?d?d

# Show cracked
hashcat -m <MODE> hashes.txt --show
hashcat -m <MODE> hashes.txt --show --username

# Resume
hashcat -m <MODE> hashes.txt rockyou.txt --restore

# Performance
hashcat -m <MODE> hashes.txt rockyou.txt --force           # Ignore warnings (GPU)
hashcat -m <MODE> hashes.txt rockyou.txt -O                # Optimized kernels
hashcat -m <MODE> hashes.txt rockyou.txt -w 3              # Workload profile (1-4)
hashcat -m <MODE> hashes.txt rockyou.txt --gpu-temp-abort=95  # Temp safety
```

### Mask Attack Character Sets

```
?l = abcdefghijklmnopqrstuvwxyz
?u = ABCDEFGHIJKLMNOPQRSTUVWXYZ
?d = 0123456789
?s = «space»!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
?a = ?l?u?d?s (all printable ASCII)
?b = 0x00 - 0xff (all bytes)

# Examples:
?u?l?l?l?d?d?d?d     = Capital + 3 lowercase + 4 digits (e.g., Pass1234)
?a?a?a?a?a?a?a?a      = 8 char all printable (very slow)
?d?d?d?d?d?d          = 6 digit PIN
?u?l?l?l?l?l?d?d?s   = Complex 9-char password
```

---

## John the Ripper

```bash
# Basic usage
john hashes.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --wordlist=rockyou.txt --rules hashes.txt

# Specific format
john --format=NT hashes.txt  # NTLM
john --format=sha512crypt hashes.txt  # Linux SHA512
john --format=krb5asrep hashes.txt   # AS-REP roast

# Show cracked
john --show hashes.txt
john --show --format=NT hashes.txt

# Resume
john --restore

# List formats
john --list=formats
john --list=formats | grep -i ntlm

# Incremental (brute force)
john --incremental hashes.txt
john --incremental=digits hashes.txt
```

### John Utilities

```bash
# Convert passwd/shadow
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt

# Convert Kerberos hashes
kirbi2john ticket.kirbi > krb_hash.txt
john --wordlist=rockyou.txt krb_hash.txt

# Convert PCAP
pcap2john traffic.pcap > pcap_hash.txt
john --wordlist=rockyou.txt pcap_hash.txt
```

---

## Hash Type Reference

| Hash Type | Hashcat Mode | John Format | Example |
|-----------|-------------|-------------|---------|
| NTLM | 1000 | NT | 32 char hex |
| MD5 | 0 | raw-md5 | 32 char hex |
| SHA1 | 100 | raw-sha1 | 40 char hex |
| SHA256 | 1400 | raw-sha256 | 64 char hex |
| SHA512 | 1700 | raw-sha512 | 128 char hex |
| bcrypt | 3200 | bcrypt | $2a$ prefix |
| SHA512crypt (Linux) | 1800 | sha512crypt | $6$ prefix |
| SHA256crypt (Linux) | 7400 | sha256crypt | $5$ prefix |
| MD5crypt (Linux) | 500 | md5crypt | $1$ prefix |
| NTLM (Domain cached) | 2100 | mscash2 | $DCC2$ prefix |
| NetNTLMv1 | 5500 | netntlm | challenge response |
| NetNTLMv2 | 5600 | netntlmv2 | challenge response |
| Kerberos AS-REP | 18200 | krb5asrep | $krb5asrep$23$ |
| Kerberos TGS RC4 | 13100 | krb5tgs | $krb5tgs$23$ |
| Kerberos TGS AES128 | 19600 | - | $krb5tgs$17$ |
| Kerberos TGS AES256 | 19700 | - | $krb5tgs$18$ |
| WPA2 | 22000 | - | PMKID/handshake |
| DPAPI | 15900 | - | DPAPI format |
| LM | 3000 | lm | 16 char hex (split) |

### Common AD Hash Types

```bash
# NTLM - most common
hashcat -m 1000 ntlm_hashes.txt rockyou.txt

# NetNTLMv2 (Responder captures)
hashcat -m 5600 netntlmv2.txt rockyou.txt

# AS-REP Roast
hashcat -m 18200 asrep.txt rockyou.txt

# Kerberoast (RC4)
hashcat -m 13100 kerberoast.txt rockyou.txt

# Kerberoast (AES256)
hashcat -m 19700 kerberoast_aes.txt rockyou.txt

# Domain Cached Credentials (DCC2)
hashcat -m 2100 dcc2.txt rockyou.txt
```

---

## Attack Modes

| Mode | Description | Flag |
|------|-------------|------|
| 0 | Dictionary (straight) | -a 0 |
| 1 | Combinator | -a 1 |
| 3 | Brute-force/mask | -a 3 |
| 6 | Hybrid dict+mask | -a 6 |
| 7 | Hybrid mask+dict | -a 7 |
| 9 | Association | -a 9 |

### Dictionary Attack (Mode 0)

```bash
hashcat -m 1000 hashes.txt rockyou.txt
hashcat -m 1000 hashes.txt rockyou.txt /usr/share/wordlists/fasttrack.txt  # Multiple wordlists
```

### Mask Attack (Mode 3)

```bash
# 8 char: upper + 6 lower + digit
hashcat -m 1000 hashes.txt -a 3 ?u?l?l?l?l?l?l?d

# Custom charset
hashcat -m 1000 hashes.txt -a 3 -1 abcdef1234 ?1?1?1?1?1?1

# Increment (try all lengths up to mask)
hashcat -m 1000 hashes.txt -a 3 ?a?a?a?a?a?a?a?a --increment --increment-min=6
```

### Hybrid Attack (Mode 6/7)

```bash
# Wordlist + mask (append digits to each word)
hashcat -m 1000 hashes.txt -a 6 rockyou.txt ?d?d?d

# Mask + wordlist (prepend)
hashcat -m 1000 hashes.txt -a 7 ?d?d?d rockyou.txt
```

---

## Wordlists and Rules

### Common Wordlists

```bash
# Kali default
/usr/share/wordlists/rockyou.txt
/usr/share/wordlists/fasttrack.txt

# SecLists (comprehensive)
# https://github.com/danielmiessler/SecLists
/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt
/usr/share/seclists/Passwords/darkweb2017-top10000.txt

# Custom AD wordlist generation
# Target's name, company, years, seasons, common patterns
```

### Generating Custom Wordlists

```bash
# CeWL - crawl website for words
cewl https://target.com -d 2 -m 6 -w cewl_words.txt

# Mentalist - rule-based wordlist generation
# (GUI tool)

# CUPP - common user passwords profiler
python3 cupp.py -i   # Interactive mode

# PACK (Password Analysis and Cracking Kit)
python3 statsgen.py rockyou.txt -o rockyou.masks  # Analyze patterns
python3 maskgen.py rockyou.masks --optindex -t 60 -o top_masks.txt
```

### Hashcat Rules

```bash
# Built-in rules (Kali)
ls /usr/share/hashcat/rules/
# Key files:
# best64.rule - good general purpose
# d3ad0ne.rule - large rule set
# dive.rule - comprehensive

# Apply single rule
hashcat -m 1000 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Multiple rules
hashcat -m 1000 hashes.txt rockyou.txt -r rule1.rule -r rule2.rule

# Generate rules with PACK
python3 rulegen.py -w rockyou.txt
```

---

## Rainbow Tables

Pre-computed hash → plaintext tables. Trade disk space for time.

```bash
# ophcrack (LM/NTLM rainbow tables)
ophcrack -g -d /path/to/tables -f hashes.txt

# RainbowCrack
rcrack /path/to/tables -l hashes.txt

# Download tables
# https://ophcrack.sourceforge.io/tables.php
# https://project-rainbowcrack.com/table.htm

# For NTLM: standard tables cover up to 8-char mixed case + digits
```

---

## Online Resources

- **CrackStation:** https://crackstation.net - MD5, SHA1, SHA256, NTLM (free, large table)
- **Hashes.com:** https://hashes.com/en/decrypt/hash - Multiple algorithm support
- **MD5Decrypt:** https://md5decrypt.net - MD5 specific
- **OnlineHashCrack:** https://www.onlinehashcrack.com - Paid, WPA/NTLM/etc

---

## Tips and Tricks

### Performance Optimization

```bash
# Check GPU status
hashcat -I
nvidia-smi  # NVIDIA

# Use GPU exclusively
hashcat -m 1000 hashes.txt rockyou.txt -D 2     # Only use GPU (device type 2)

# Multiple GPUs
hashcat -m 1000 hashes.txt rockyou.txt -d 1,2   # Specific GPU devices

# Increase performance
hashcat -m 1000 hashes.txt rockyou.txt -O -w 4  # Optimized + max workload

# Cloud GPU (AWS p3.2xlarge, Google Cloud GPU instances)
# Much faster than local CPU
```

### Cracking Strategy

```bash
# 1. Dictionary first (fastest)
hashcat -m 1000 hashes.txt rockyou.txt

# 2. Dictionary + best64 rules
hashcat -m 1000 hashes.txt rockyou.txt -r best64.rule

# 3. Multiple wordlists
hashcat -m 1000 hashes.txt rockyou.txt fasttrack.txt darkweb2017.txt

# 4. Targeted mask (if you know pattern)
hashcat -m 1000 hashes.txt -a 3 ?u?l?l?l?l?d?d?d  # Company1234

# 5. PACK-generated masks (if you've analyzed cracked hashes from similar breach)

# 6. Combinator
hashcat -m 1000 hashes.txt -a 1 words.txt words.txt

# 7. Large mask attack (last resort, very slow)
hashcat -m 1000 hashes.txt -a 3 -i ?a?a?a?a?a?a?a?a  # Up to 8 chars all
```

### AD-Specific Tips

```bash
# Company + year pattern (very common)
echo "Company2023\nCompany2024\nCompany!\nCompany@2023" > company_words.txt
hashcat -m 1000 hashes.txt company_words.txt -r best64.rule

# Seasons + year
echo "Spring2024!\nSummer2024!\nFall2024!\nWinter2024!\nSpring2024" > seasons.txt
hashcat -m 1000 hashes.txt seasons.txt

# Welcome/Password patterns
hashcat -m 1000 hashes.txt -a 3 Welcome?d?d?d?d
hashcat -m 1000 hashes.txt -a 3 Password?d?d?d?d?s
hashcat -m 1000 hashes.txt -a 3 P@ssw?d?d?d
```
