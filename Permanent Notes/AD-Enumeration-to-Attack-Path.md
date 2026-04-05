---
title: "AD Enumeration — From Enumeration to Attack Path"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - active-directory
  - enumeration
  - bloodhound
  - methodology
  - oscp
---

# AD Enumeration — From Enumeration to Attack Path

**Core idea:** AD enumeration is about building a map, not taking action. The goal is to find the chain: *what you own → what you can reach → what leads to Domain Admin*. Rushing to exploit the first thing you find is how you get caught and miss better paths.

---

## The Assumed Breach Mental Model

You start as a low-privilege domain user (stephanie). AD contains thousands of objects. The question is: **what relationships exist between objects that create an exploitable path?**

Three types of relationships matter most:
1. **Access rights** — who has admin where (`AdminTo`)
2. **Sessions** — who is logged in where (`HasSession`)
3. **Permissions** — who has control over what AD objects (`GenericAll`, `WriteDACL`, etc.)

When these three overlap, you have an attack path.

---

## The Layered Enumeration Approach

Work from fast/broad to slow/specific:

```
1. net.exe          → 30 seconds, works everywhere, misses nested groups
2. PowerView        → minutes, full picture including nesting and attributes
3. SharpHound       → minutes, captures everything BloodHound can analyze
4. BloodHound       → find shortest path you'd have missed manually
```

**Don't skip step 1** — even on a hardened network, `net user /domain` always works and gives you an immediate target list. Look for accounts with admin/svc/service in the name immediately.

---

## net.exe vs PowerView — The Nested Group Blind Spot

`net group "Sales Department" /domain` → showed pete and stephanie only.

PowerView revealed: Sales Department ← Development Department ← Management Department ← jen.

**The lesson:** net.exe only shows user objects as group members. PowerView shows both user and group objects, uncovering nested membership chains that net.exe hides. Always follow nested groups to their end — the deepest user in the chain may be the most privileged.

---

## The Session + Admin Overlap Pattern

The most powerful attack path in AD is:

```
[Your user] --AdminTo--> [Machine A] --HasSession--> [Privileged user]
```

This means:
- You can log into Machine A (admin access)
- A privileged user has credentials cached on Machine A
- You can steal those credentials (Mimikatz / Kiwi)

**How to find this manually:**
1. `Find-LocalAdminAccess` → where do you have admin?
2. `.\PsLoggedOn.exe \\MACHINE` → who is logged in there?
3. Cross-reference: if a DA (or high-value account) has a session on a machine you admin → that's your path

**Why PsLoggedOn over Get-NetSession:** NetSessionEnum is blocked on modern Windows (since Win10 1709 / Server 2019 1809) by the `SrvsvcSessionInfo` registry key. PsLoggedOn reads `HKEY_USERS` directly via Remote Registry service — more reliable, but requires Remote Registry to be enabled.

---

## The Three Share Secrets

AD shares routinely expose credentials through:

1. **SYSVOL GPP passwords** — `old-policy-backup.xml` with `cpassword` field → decrypt with `gpp-decrypt` on Kali. AES-256 but the key is public on MSDN.

2. **Cleartext in "private" folders** — "do-not-share" folders that are shared. Email archives. Setup scripts. Search for: passwords, credentials, config, setup.

3. **Scripts with hardcoded credentials** — Logon scripts, deployment scripts stored in NETLOGON or SYSVOL.

**Always enumerate SYSVOL first** — it's world-readable and frequently has GPP leftovers that admins forgot.

---

## ACL Misconfigs — GenericAll is the Golden Ticket

If a regular domain user has `GenericAll` on any object, that's almost certainly a misconfiguration. What it enables:

- **GenericAll on a group** → add yourself → inherit group's permissions
- **GenericAll on a user** → reset their password → take over their account
- **GenericAll on a computer** → create an RBCD attack (Resource-Based Constrained Delegation)

Find it:
```powershell
Get-ObjectAcl -Identity "TARGET" | where {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier
# Convert SIDs → look for non-admin accounts
```

---

## BloodHound Workflow — Mark → Query → Path

The three-step BloodHound workflow:

1. **Mark owned principals** — right-click → Mark as Owned (start with your user)
2. **Run "Shortest Paths to DA from Owned Principals"** — shows only paths reachable from what you control
3. **Right-click each edge → Help** — BloodHound explains exactly how to abuse the relationship

Don't just look at the shortest path. Look at ALL paths. A longer path may be safer (less detection) than the shortest one.

---

## Connections

- [[PWK-Module-21-Active-Directory-Enumeration]] — full commands and walkthroughs
- [[PWK-Module-22-Attacking-AD-Authentication]] — Kerberoasting SPN accounts found in enumeration
- [[Windows-PrivEsc-Decision-Tree]] — once inside a machine, PrivEsc to harvest credentials
- [[Pivoting-Mental-Model]] — lateral movement between machines to reach cached DA credentials

---

*From: PWK Module 21*
