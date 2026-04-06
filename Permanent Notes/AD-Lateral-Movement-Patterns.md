---
title: "AD Lateral Movement — Pattern Recognition"
created: "2026-04-06"
type: permanent-note
status: active
tags:
  - permanent-note
  - active-directory
  - lateral-movement
  - pass-the-hash
  - kerberos
  - golden-ticket
  - methodology
  - oscp
---

# AD Lateral Movement — Pattern Recognition

**Core idea:** Lateral movement in AD is a credential relay problem. You acquire credentials (plaintext, hash, or ticket) on one machine and spend them on another. The technique you choose depends on what form the credential is in and what authentication protocol the target will accept.

---

## The Credential-to-Technique Map

| What you have | Best technique | Tool |
|---------------|---------------|------|
| Plaintext password | WMI / WinRM / PsExec | wmic, winrs, PsExec64 |
| NTLM hash, target accepts NTLM | Pass the Hash | impacket-wmiexec -hashes |
| NTLM hash, want Kerberos | Overpass the Hash | Mimikatz sekurlsa::pth |
| Active TGS/TGT in memory | Pass the Ticket | Mimikatz sekurlsa::tickets /export → kerberos::ptt |
| krbtgt hash (post-DC compromise) | Golden Ticket | Mimikatz kerberos::golden |
| Service account hash | Silver Ticket | Mimikatz kerberos::golden /service |
| DA on DC | Shadow Copy → NTDS.dit | vshadow + impacket-secretsdump LOCAL |

---

## The Hostname vs IP Rule (Critical)

**Always use hostnames for Kerberos-based lateral movement, never IPs.**

When you specify an IP address, Windows cannot resolve the SPN (Service Principal Name) needed for Kerberos, so it falls back to NTLM. This breaks:
- Overpass the Hash (your TGT is useless if NTLM is used)
- Golden Ticket (forged TGT is ignored if NTLM is forced)
- Pass the Ticket (same)

```
.\PsExec.exe \\FILES04 cmd      ← Kerberos (hostname) — uses your TGT ✓
.\PsExec.exe \\192.168.50.73    ← NTLM (IP) — ignores TGT, fails if no hash ✗
```

This one rule catches many "why didn't it work?" moments in AD labs.

---

## Overpass the Hash vs Pass the Ticket

Both use Kerberos. They're different stages of the same pipeline:

**Overpass the Hash** = hash → TGT (you start fresh — no existing ticket needed)
**Pass the Ticket** = steal existing TGT/TGS → inject into your session (no hash needed)

Use Overpass the Hash when you have a hash but no active session from that user.
Use Pass the Ticket when the target user is already logged in somewhere you have admin access — steal their ticket directly, no cracking required.

---

## Golden Ticket = Domain Persistence, Not Just Access

A Golden Ticket is fundamentally different from other techniques — it's persistence, not just movement. Because it's signed with `krbtgt`, it:

- Grants access to **every service** in the domain
- **Never expires** (you set the validity period — default 10 years)
- **Survives password resets** of normal user accounts
- Works from **any machine**, even non-domain-joined

**The only way to invalidate it:** Reset the `krbtgt` password **twice** (because the previous hash is also cached). Most organizations never do this because it temporarily disrupts Kerberos authentication domain-wide.

**What this means operationally:** Once you have the krbtgt hash via DCSync or DC compromise, document it carefully. You can regenerate a Golden Ticket months later if you lose access.

---

## Shadow Copy — The Clean Endgame

Shadow Copy + impacket-secretsdump LOCAL is the cleanest way to dump all domain credentials:

1. No LSASS access on individual machines (lower detection risk)
2. Extracts hashes for **every** account including disabled/service accounts
3. Offline — no network traffic during extraction (just local file ops on DC)
4. Provides NTLM hashes, Kerberos AES keys, and more

The two required files: `ntds.dit` (the database) + `SYSTEM` hive (the decryption key). Without the SYSTEM hive, `ntds.dit` is encrypted and unreadable.

---

## The WMI/WinRM/PsExec Triangle

All three execute commands remotely using plaintext creds. Pick based on OpSec:

| Tool | Detection footprint | Notes |
|------|--------------------|----|
| WMI (CimSession) | Low — uses DCOM/RPC | No service created; blends with normal WMI traffic |
| WinRM | Medium — WinRM logs | Port 5985/5986; PowerShell remoting logs exist |
| PsExec | High — creates PSEXESVC service | Event ID 7045 logged; well-known by defenders |

In a real engagement, prefer WMI or WinRM over PsExec when stealth matters. PsExec is fine in OSCP lab scenarios.

---

## Connections

- [[PWK-Module-23-Lateral-Movement-AD]] — full commands and walkthroughs
- [[AD-Authentication-Attack-Patterns]] — where the hashes and tickets come from
- [[AD-Enumeration-to-Attack-Path]] — BloodHound paths that identify lateral movement targets
- [[Pivoting-Mental-Model]] — when lateral movement requires tunneling through intermediate hosts

---

*From: PWK Module 23*
