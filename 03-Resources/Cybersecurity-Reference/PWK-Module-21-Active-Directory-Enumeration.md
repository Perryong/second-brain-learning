---
title: "PWK Module 21 — Active Directory Introduction and Enumeration"
created: "2026-04-05"
type: reference-note
status: active
tags:
  - reference-note
  - active-directory
  - enumeration
  - powerview
  - bloodhound
  - sharphound
  - oscp
source: "PWK Module 21 — Active Directory Introduction and Enumeration"
---

# PWK Module 21 — Active Directory Introduction and Enumeration

## Overview

AD enumeration starts with an **assumed breach** — you have a low-privilege domain user (e.g., `stephanie`) and you want to map the domain to find attack paths. The goal is building a picture of users, groups, computers, sessions, SPNs, permissions (ACLs), and shares before attempting exploitation.

**Lab context:** Domain is `corp.com`, DC is `DC1.corp.com`, attacker machine is `CLIENT75`, compromised user is `stephanie`.

---

## 1. Legacy Windows Tools (net.exe)

Quick, built-in enumeration that works anywhere — but limited.

```cmd
net user /domain                          # list all domain users
net user jeffadmin /domain                # inspect specific user (group memberships, last logon)
net group /domain                         # list all domain groups
net group "Sales Department" /domain      # list members of a specific group
```

**Limitation:** `net.exe` only shows user objects, not group objects inside groups (misses nested groups). Can't display specific object attributes.

---

## 2. PowerShell + .NET LDAP Enumeration

Build a reusable LDAP search script manually. Good for understanding what PowerView does under the hood.

### Building the LDAP Path

```powershell
# Get PDC hostname dynamically
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$PDC = $domainObj.PdcRoleOwner.Name

# Get Distinguished Name via ADSI
$DN = ([adsi]'').distinguishedName

# Assemble full LDAP path
$LDAP = "LDAP://$PDC/$DN"
```

### Search All Domain Objects

```powershell
# Filter by samAccountType
# 805306368 (0x30000000) = user objects
# objectClass=group = group objects
$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter = "(samAccountType=805306368)"   # users
$result = $dirsearcher.FindAll()

# Print all properties for each object
foreach ($obj in $result) {
    foreach ($prop in $obj.Properties) { $prop }
    Write-Host "------"
}
```

### Reusable LDAPSearch Function

```powershell
function LDAPSearch {
    param ([string]$LDAPQuery)
    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DN = ([adsi]'').distinguishedName
    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DN")
    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)
    return $DirectorySearcher.FindAll()
}

# Usage
LDAPSearch -LDAPQuery "(samAccountType=805306368)"   # all users
LDAPSearch -LDAPQuery "(objectClass=group)"           # all groups

# Enumerate groups with members
foreach ($group in $(LDAPSearch -LDAPQuery "(objectClass=group)")) {
    $group.properties | select {$_.cn}, {$_.member}
}
```

**Run with:**
```cmd
powershell -ep bypass -File .\enumeration.ps1
```

**Key finding from this approach:** Net.exe showed `Sales Department` had pete and stephanie. LDAPSearch revealed `Development Department` was also a nested member → unraveled chain: Sales → Development → Management → jen.

---

## 3. PowerView Enumeration

**Location:** `C:\Tools\PowerView.ps1`

```powershell
Import-Module .\PowerView.ps1    # or: . .\PowerView.ps1
```

### Users and Groups

```powershell
Get-NetDomain                                        # domain info, PDC, forest
Get-NetUser | select cn                              # list all usernames
Get-NetUser | select cn,pwdlastset,lastlogon         # dormant account detection
Get-NetUser -SPN | select samaccountname,serviceprincipalname  # SPN accounts

Get-NetGroup | select cn                             # list all groups
Get-NetGroup "Sales Department" | select member      # group members (shows nested groups!)
```

**Advantage over net.exe:** PowerView shows nested group members (group objects within groups), allows piping into `select` for specific attributes.

### Computers and OS

```powershell
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
# Identify: oldest OS, web servers, file servers, DCs
```

---

## 4. Finding Admin Access and Logged-on Users

### Where Do We Have Admin?

```powershell
Find-LocalAdminAccess
# Sprays OpenServiceW (SC_MANAGER_ALL_ACCESS) against all domain computers
# Returns computers where current user has local admin
# Example output: CLIENT74.corp.com
```

### Who Is Logged On Where?

**Problem:** NetSessionEnum is restricted on modern Windows (since Windows 10 build 1709 and Server 2019 build 1809). The `SrvsvcSessionInfo` registry key limits remote enumeration.

```powershell
Get-NetSession -ComputerName FILES04 -Verbose
# May return "Access is denied" on modern Windows
```

**Solution: PsLoggedOn (SysInternals)**

PsLoggedOn reads `HKEY_USERS` for logged-in SIDs and uses Remote Registry service. Requires Remote Registry to be enabled (default on: servers; disabled on: workstations since Win8).

```cmd
.\PsLoggedOn.exe \\FILES04      # jeff is logged in via domain account
.\PsLoggedOn.exe \\WEB04        # no users logged in
.\PsLoggedOn.exe \\CLIENT74     # jeffadmin has an open session!
```

**Why this matters:** If `stephanie` has admin on `CLIENT74` and `jeffadmin` has a session there, we can log in to `CLIENT74` and steal `jeffadmin`'s credentials → Domain Admin.

---

## 5. SPN Enumeration (Service Accounts)

SPNs (Service Principal Names) link service accounts to services. Accounts with SPNs are service accounts and typically have elevated privileges. They are targets for **Kerberoasting**.

```cmd
# Built-in Windows tool
setspn -L iis_service            # list SPNs for specific account
```

```powershell
# PowerView
Get-NetUser -SPN | select samaccountname,serviceprincipalname
# Output: iis_service → HTTP/web04.corp.com:80, HTTP/web04.corp.com:443
# krbtgt = Kerberos Ticket-Granting Ticket account (DC's KDC service)
```

**Note the hostname:**
```cmd
nslookup web04.corp.com          # resolve to internal IP for later access
```

Document SPN accounts — they become targets in Module 22 (Kerberoasting).

---

## 6. Object Permission Enumeration (ACLs)

ACLs define who can do what to each AD object. Misconfigurations create privilege escalation paths.

### Key Dangerous Permissions

| Permission | What an attacker can do |
|-----------|------------------------|
| `GenericAll` | Full control — add members, reset passwords, modify attributes |
| `GenericWrite` | Write to any non-protected attribute |
| `WriteOwner` | Change object owner → then grant yourself GenericAll |
| `WriteDACL` | Modify ACL → grant yourself any permission |
| `AllExtendedRights` | Reset passwords, Kerberoast, etc. |
| `ForceChangePassword` | Reset user's password without knowing current one |
| `Self` | Add yourself to a group |

### Enumerate ACEs

```powershell
# List all ACEs for an object
Get-ObjectAcl -Identity stephanie

# Key fields: ActiveDirectoryRights, ObjectSID (target object), SecurityIdentifier (who has access)

# Convert SID to readable name
Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-1104

# Find all GenericAll permissions on a group
Get-ObjectAcl -Identity "Management Department" | where {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights

# Convert multiple SIDs at once
"S-1-5-21-...1104","S-1-5-21-...512" | Convert-SidToName
```

### Abusing GenericAll

If `stephanie` has `GenericAll` on `Management Department` group:

```cmd
# Add ourselves to the group
net group "Management Department" stephanie /add /domain

# Verify
Get-NetGroup "Management Department" | select member

# Clean up (remove ourselves)
net group "Management Department" stephanie /del /domain
```

**Real-world value:** GenericAll on a group → add yourself → inherit all that group's permissions and access. GenericAll on a user → force password reset → take over account.

---

## 7. Domain Share Enumeration

Shares often contain credentials, config files, and sensitive documents.

```powershell
Find-DomainShare                           # list all shares in domain
Find-DomainShare -CheckShareAccess         # only shares accessible to current user
```

### SYSVOL — Always Check

SYSVOL is world-readable by all domain users. Contains GPO scripts and config.

```cmd
ls \\DC1.corp.com\SYSVOL\corp.com\Policies\
ls \\DC1.corp.com\SYSVOL\corp.com\Policies\oldpolicy\
type \\DC1.corp.com\SYSVOL\corp.com\Policies\oldpolicy\old-policy-backup.xml
```

### GPP Password Decryption (cpassword)

Group Policy Preferences (GPP) stored passwords encrypted with AES-256, but Microsoft published the private key on MSDN. Any cpassword value in SYSVOL is crackable.

```bash
# On Kali
gpp-decrypt "+bm3FzPzrTNx5vCG..."
# Output: cleartext password
```

### Non-Default Shares

```cmd
ls \\FILES04.corp.com\docshare\
# Look for: scripts, config files, documents
type \\FILES04.corp.com\docshare\do-not-share\start-email.txt
# Found: HenchmanPutridBonbon11! (plaintext password for jeff)
```

**Pattern:** "do-not-share" folders that are actually shared. Forgotten email archives. Setup scripts with embedded credentials.

---

## 8. SharpHound — Automated Data Collection

SharpHound automates the collection of all enumerable AD data and packages it for BloodHound.

```powershell
# On compromised Windows machine (CLIENT75)
Import-Module .\SharpHound.ps1

Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\stephanie\Desktop -OutputPrefix "corp audit"
# Collects: users, groups, computers, sessions, ACLs, GPOs, domain trusts
# Outputs: corp audit_<timestamp>_BloodHound.zip + .bin cache
# Delete the .bin file — not needed for analysis
```

Transfer the ZIP to Kali (scp, Python web server, etc.).

---

## 9. BloodHound — Graph Analysis

BloodHound visualizes AD relationships as a graph — nodes (objects) connected by edges (relationships).

### Setup on Kali

```bash
sudo neo4j start                   # start Neo4j graph database
# Browse http://localhost:7474 → authenticate (neo4j:neo4j or your password)
bloodhound &                        # launch BloodHound GUI
```

Authenticate to Neo4j DB → Upload ZIP from SharpHound (drag-and-drop or Upload Data button).

### Key Analysis Queries

| Query | What it shows |
|-------|--------------|
| Find all Domain Admins | Who is DA — jeffadmin + Administrator |
| Find Shortest Paths to Domain Admins | All attack paths to DA |
| Shortest Paths to DA from Owned Principals | Paths from your controlled objects |

### Reading the Graph

- **Nodes** = AD objects (users, computers, groups)
- **Edges** = relationships between nodes
- **Right-click edge → Help** = BloodHound explains the relationship and how to abuse it

**Key edge types:**

| Edge | Meaning |
|------|---------|
| `MemberOf` | User/group is member of a group |
| `AdminTo` | Has local admin on that computer |
| `HasSession` | User has an active session on that computer |
| `GenericAll` | Full control over target object |
| `CanRDP` | Can RDP to that computer |

### Mark Owned Principals

Right-click a node → "Mark as Owned" → then run "Shortest Paths to DA from Owned Principals" to see realistic attack paths.

### Key Attack Path Discovery (Lab Example)

```
stephanie (owned) → AdminTo → CLIENT74
jeffadmin → HasSession → CLIENT74
jeffadmin → MemberOf → Domain Admins
```

**Translation:** Log into CLIENT74 as stephanie (local admin) → steal jeffadmin's cached credentials → use as Domain Admin.

---

## Quick Enumeration Checklist (Assumed Breach)

```powershell
# 1. Domain basics
Get-NetDomain
net user /domain | findstr "admin\|svc\|service"

# 2. Users (dormant, privileged)
Get-NetUser | select cn,pwdlastset,lastlogon
Get-NetUser -SPN | select samaccountname,serviceprincipalname    # Kerberoasting targets

# 3. Groups + nesting
Get-NetGroup | select cn
Get-NetGroup "Domain Admins" | select member

# 4. Computers + OS
Get-NetComputer | select dnshostname,operatingsystem

# 5. Where do we have admin?
Find-LocalAdminAccess

# 6. Who is logged in where?
.\PsLoggedOn.exe \\COMPUTERNAME

# 7. ACL misconfigs
Get-ObjectAcl -Identity "GROUP" | where {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier

# 8. Shares
Find-DomainShare -CheckShareAccess
ls \\DC\SYSVOL                                    # GPP passwords

# 9. Automate everything
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\user\Desktop
```

---

## Connections

- [[PWK-Module-22-Attacking-AD-Authentication]] — Kerberoasting SPN accounts found here; AS-REP roasting; Pass-the-Hash
- [[AD-Enumeration-to-Attack-Path]] — mental model for turning enumeration findings into attack chains
- [[PWK-Module-16-Password-Attacks]] — GPP passwords → crack with gpp-decrypt; hash cracking
- [[Pivoting-Mental-Model]] — AD often requires lateral movement between compromised hosts

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [[Active Directory Basics]] | TryHackMe | Domain structure, users/groups/OUs, GPOs, trusts — foundational AD concepts |
| [[Attacking Kerberos]] | TryHackMe | Kerbrute user enumeration, AS-REP Roasting, Kerberoasting setup and context |
| [[VulnNet Roasted]] | TryHackMe | Real AD domain: full enumeration pipeline from null session → SMB shares → credential discovery |
| [[RazorBlack]] | TryHackMe | AD enumeration leading into authentication attacks |

---

*From: PWK Module 21*
