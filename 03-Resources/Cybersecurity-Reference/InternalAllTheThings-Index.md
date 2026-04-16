---
title: "InternalAllTheThings Reference Index"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - index
  - active-directory
  - red-team
  - oscp
---

# InternalAllTheThings Reference Index

Master index of all reference notes imported from [swisskyrepo/InternalAllTheThings](https://github.com/swisskyrepo/InternalAllTheThings), organized by attack phase and topic.

Cross-links to PWK modules:
- [[PWK-Module-21-Active-Directory-Enumeration]]
- [[PWK-Module-22-Attacking-AD-Authentication]]
- [[PWK-Module-23-Lateral-Movement-AD]]
- [[PWK-Module-24-Assembling-the-Pieces]]
- [[THM-HTB-Practice-Labs-Index]]

---

## Active Directory

| Note | What it covers |
|------|---------------|
| [[AD-Enumeration-Reference]] | PowerView, LDAP queries, BloodHound, net commands, ACL/ACE enumeration, Linux AD (ldapsearch, kinit, keytab) |
| [[AD-ADCS-Certificate-Services]] | ESC1–ESC8 misconfigs, certipy, certutil, certificate template abuse, NTLM relay to ADCS |
| [[AD-Kerberos-Attacks]] | AS-REP roasting, Kerberoasting, unconstrained/constrained/RBCD delegation, S4U2Self/Proxy, Golden/Silver/Diamond tickets, Bronze Bit |
| [[AD-Credential-Attacks]] | Pass-the-Hash, Overpass-the-Hash, Pass-the-Key, NTDS dumping (secretsdump/DCSync/VSS), LAPS, GMSA, Shadow Credentials, GPP passwords, DSRM, password spraying |
| [[AD-Trust-and-Forest-Attacks]] | Cross-forest trust attacks, SID history injection, inter-realm tickets, PAM trust abuse |
| [[AD-MITM-Relay-Attacks]] | Responder (LLMNR/NBT-NS poisoning), NTLM relay (ntlmrelayx), ADCS relay (ESC8), PetitPotam, PrinterBug, coercion attacks |
| [[AD-CVE-Reference]] | ZeroLogon (CVE-2020-1472), NoPAC (CVE-2021-42278/42287), PrintNightmare (CVE-2021-1675), PrivExchange, MS14-068 |
| [[Active-Directory-Cheatsheet]] | OSCP-focused quick reference: enumeration, Mimikatz, PtH/OPtH/PtT, Silver Tickets, DCOM, hash cracking |

---

## Credentials & Hashes

| Note | What it covers |
|------|---------------|
| [[Mimikatz-Reference]] | Complete Mimikatz command reference: sekurlsa, kerberos, lsadump, token, privilege, dpapi modules |
| [[Hash-Cracking-Reference]] | Hashcat modes (1000/5600/13100/18200/etc.), john usage, rule files, wordlists, rainbow tables |

---

## Privilege Escalation

| Note | What it covers |
|------|---------------|
| [[Windows-Privilege-Escalation-Reference]] | Service misconfigs, unquoted paths, AlwaysInstallElevated, token impersonation (Potato attacks), UAC bypass, scheduled tasks, registry keys, DLL hijacking |
| [[Linux-Privilege-Escalation-Reference]] | SUID/SGID, sudo misconfigs, cron jobs, capabilities, writable /etc/passwd, NFS no_root_squash, PATH hijacking, Docker socket, kernel exploits |

---

## Persistence

| Note | What it covers |
|------|---------------|
| [[Windows-Persistence-Reference]] | Registry run keys, scheduled tasks, WMI event subscriptions, services, DLL hijacking, Startup folder, IFEO, COM hijacking, Golden/Silver tickets |
| [[Linux-Persistence-Reference]] | Cron jobs, SSH authorized_keys, bashrc/.profile, PAM backdoors, systemd services, SUID shells, LD_PRELOAD |

---

## Evasion

| Note | What it covers |
|------|---------------|
| [[AV-EDR-Evasion-Reference]] | AMSI bypass (PowerShell reflection, .NET, patching), EDR evasion, process injection, obfuscation, ETW patching |

---

## Lateral Movement & Access

| Note | What it covers |
|------|---------------|
| [[Network-Pivoting-Reference]] | SSH local/remote/dynamic tunneling, chisel, ligolo-ng, socat, proxychains, Meterpreter routes, sshuttle, double-pivoting |
| [[Initial-Access-Techniques]] | Infection chain model, container formats (ISO/ZIP/WIM), payload types, LOLBins (certutil/mshta/rundll32/bitsadmin), Windows remote exec (netexec/impacket/RDP/WinRM/WMI) |
| [[PowerShell-Red-Team-Reference]] | Execution policy bypass, download cradles, reverse shells, AMSI one-liners, PSRemoting, credential objects |

---

## Recon & Infrastructure

| Note | What it covers |
|------|---------------|
| [[Network-Discovery-Reference]] | DHCP/DNS/NBT-NS/ARP/MDNS discovery, nmap scan types and scripts, masscan, ping sweeps, Responder passive recon, MITM (Bettercap, SSL MITM) |
| [[Container-Security-Reference]] | Docker escapes (socket, open API, privileged/cgroup, runC CVE, kernel module), Kubernetes RBAC abuse, SA token theft, malicious pods, kubelet API, etcd enumeration |

---

## Quick Attack Path Reference

### "I have a foothold on a Windows domain-joined machine"
1. [[AD-Enumeration-Reference]] — enumerate domain (PowerView, BloodHound)
2. [[AD-Kerberos-Attacks]] — AS-REP roast / Kerberoast service accounts
3. [[AD-Credential-Attacks]] — dump credentials (Mimikatz, NTDS)
4. [[AD-Kerberos-Attacks]] — escalate with delegation or Golden/Silver ticket
5. [[AD-Trust-and-Forest-Attacks]] — cross-forest if applicable

### "I can reach a machine on a different subnet"
→ [[Network-Pivoting-Reference]] — set up SOCKS proxy / tunnel

### "I have local admin but not DA"
→ [[AD-Credential-Attacks]] — PtH/OPtH, DCSync
→ [[AD-MITM-Relay-Attacks]] — coerce DC auth, relay to LDAP/SMB

### "I see a certificate authority in the domain"
→ [[AD-ADCS-Certificate-Services]] — check ESC1–ESC8

### "I'm inside a Docker container"
→ [[Container-Security-Reference]] — check for mounted socket, privileged flags, cgroup escape

### "I need to maintain access"
→ [[Windows-Persistence-Reference]] or [[Linux-Persistence-Reference]]
→ [[AD-Kerberos-Attacks]] — Golden Ticket (requires KRBTGT hash)
