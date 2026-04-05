---
title: "AD Authentication Attacks — Pattern Recognition"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - active-directory
  - kerberos
  - kerberoasting
  - as-rep-roasting
  - dcsync
  - methodology
  - oscp
---

# AD Authentication Attacks — Pattern Recognition

**Core idea:** Every AD authentication attack exploits one of three things: (1) passwords that can be guessed, (2) tickets that can be forged because you have a hash, (3) hashes that can be extracted because you have sufficient access. The attack you choose depends entirely on what access you already have.

---

## The Three Kerberos Attacks — Compared

| Attack | What you need | What you get | Where the weakness is |
|--------|--------------|--------------|----------------------|
| AS-REP Roasting | Any domain user (or none) | AS-REP hash to crack offline | Account has DONT_REQ_PREAUTH enabled |
| Kerberoasting | Any valid domain credentials | TGS-REP hash to crack offline | Service account has weak password |
| Silver Ticket | NTLM hash of service account | Forged TGS for any user, any permission | Service trusts its own hash-signed ticket |

**The key difference:** AS-REP and Kerberoasting require offline cracking (hash → password). Silver Ticket doesn't — you forge the ticket directly from the hash. If cracking fails, Silver Ticket still works.

---

## Why Kerberoasting Is High-Value

Every domain user can request a TGS for any SPN. This is by design — it's how Kerberos works. The vulnerability isn't a bug; it's that service accounts often have weak, human-set passwords.

**The attack in one line:** "Request a TGS for HTTP/web04.corp.com → the KDC gives you a blob encrypted with iis_service's NTLM hash → crack offline."

**What makes it reliable:**
- No lockout risk — you're requesting tickets, not failing logins
- Offline cracking — no network noise during the crack phase
- Any authenticated user can do it — no admin required

**When it fails:** Group Managed Service Accounts (gMSA) — 120-character random passwords. If you see `$` at the end of the service account name, it's likely gMSA → skip it.

---

## The Silver Ticket Insight — Forging Without the KDC

The KDC is not involved in validating a Silver Ticket at service access time. The service account's NTLM hash is both the signing key and the verification key. If you have the hash, you can sign a ticket yourself that the service will accept.

**What this means practically:**
- You can assign yourself to Domain Admins in the ticket even if you're not
- You can use any username (even fake ones)
- No DC traffic during ticket use — harder to detect
- Persists as long as the service account hash doesn't change

**Trade-off vs Kerberoasting:** Kerberoasting gives you the cleartext password (useful for lateral movement). Silver Ticket gives you service access without the password (useful when cracking fails, but scoped to one service).

---

## DCSync — The Endgame Dump

DCSync is less of an attack and more of a capability — when you reach Domain Admin, you can harvest every user's NTLM hash from the domain. No need to touch LSASS on individual machines.

**Why it's cleaner than Mimikatz:** DCSync uses a legitimate AD replication API (DRSUAPI). It generates DC-to-DC replication traffic, which is normal in multi-DC environments. LSASS access on workstations is much more suspicious.

**The krbtgt hash is the holy grail:** DCSync can dump the `krbtgt` account hash, which enables Golden Tickets (forged TGTs that grant access to everything). This is covered in Module 23.

---

## Password Spraying — Lockout Calculation

Always do this math before spraying:

```
net accounts → Lockout threshold: 5 → safe = 4 attempts per window
             → Lockout observation window: 30 min
             → 4 × (1440/30) = 192 attempts per user per day
```

**Tool selection by noise level (quietest → loudest):**
1. **kerbrute** — 2 UDP frames per attempt, no SMB, minimal logging
2. **Spray-Passwords.ps1** — LDAP connection, moderate noise
3. **crackmapexec smb** — full SMB connection per attempt, very noisy

**crackmapexec bonus:** `Pwn3d!` in output = that user has local admin on the target. This immediately tells you where you can run Mimikatz next.

---

## The Hash-to-Access Pipeline

Once you have any NTLM hash:

```
NTLM hash
    ├── Crack offline (hashcat -m 1000)
    │      → cleartext password → password spray / lateral movement
    ├── Pass-the-Hash (Module 23)
    │      → authenticate to services without cracking
    └── Use for ticket forgery
           → Silver Ticket (service account hash)
           → Golden Ticket (krbtgt hash)
```

The hash is often more valuable than the password — many lateral movement techniques work directly with NTLM hashes.

---

## Connections

- [[PWK-Module-22-Attacking-AD-Authentication]] — full commands and walkthroughs
- [[AD-Enumeration-to-Attack-Path]] — finding SPN accounts and vulnerable users before attacking
- [[PWK-Module-23-Lateral-Movement-AD]] — using hashes from DCSync for Pass-the-Hash and Golden Tickets
- [[Password-Cracking-Methodology]] — hashcat modes (1000=NTLM, 18200=AS-REP, 13100=TGS)

---

*From: PWK Module 22*
