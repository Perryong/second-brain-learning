---
title: "Windows PrivEsc — Decision Tree and Mental Model"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - windows
  - privilege-escalation
  - methodology
  - oscp
---

# Windows PrivEsc — Decision Tree and Mental Model

**Core idea:** Windows PrivEsc isn't random — it follows a predictable hierarchy. Start with the fastest wins (credentials in files, history, transcripts), then move to misconfigurations (writable service binaries, DLL hijacking), then automation (WinPEAS/PowerUp), then kernel exploits as a last resort. The key insight is that Windows systems are full of credentials left in plaintext by administrators who think they're being secure.

---

## The Priority Order

```
1. Credentials in files / history / transcripts  → fastest, no exploit needed
2. Weak service permissions (binary hijacking)   → reliable, survives reboots
3. DLL hijacking                                 → requires process restart
4. Token impersonation (Potato attacks)          → needs SeImpersonatePrivilege
5. UAC bypass                                    → already in Admins group
6. Unquoted service paths                        → space in path without quotes
7. Scheduled tasks with writable scripts         → like cron abuse on Linux
8. Kernel exploit                                → last resort, unstable
```

---

## The Integrity Level Trap

The most common confusion: **being in the Administrators group does not mean you have admin privileges right now.**

Windows gives admin users two tokens simultaneously:
- **Medium integrity token** — used by default for all operations
- **High integrity token** — only activated after UAC consent

If your shell is Medium integrity, `whoami /groups` still shows `Administrators` — but you can't write to `C:\Windows\System32`, install services, or do most admin things.

**How to check:**
```powershell
whoami /groups | findstr "Integrity"
# Medium Mandatory Level → need UAC bypass to reach High
# High Mandatory Level  → already elevated
```

**So the correct question isn't "am I an admin?" — it's "am I running at High integrity?"**

---

## Credential Hunting Priority

Windows machines leak credentials constantly. Check in this order:

```
1. PSReadLine history     → C:\Users\%USER%\AppData\Roaming\...\ConsoleHost_history.txt
2. PowerShell transcripts → C:\Users\Public\Transcripts\
3. XAMPP / web app configs → C:\xampp\passwords.txt, my.ini, web.config
4. User home files        → *.txt, *.pdf, *.docx on Desktop and Documents
5. KeePass databases      → *.kdbx anywhere on C:\
6. Registry               → winpeas shows stored credentials in registry hives
7. Unattend.xml           → C:\Windows\Panther\ — sysprep files with admin passwords
```

Real lesson: **one found password unlocks a chain.** In the module example:
- asdf.txt → steve's password
- steve can read my.ini → backupadmin's password
- backupadmin is in Administrators → escalated

---

## Service Binary Hijacking — The One-Page Summary

```
Get-CimInstance win32_service → find services running as LocalSystem
icacls <service_binary>       → check if BUILTIN\Users has (F) or (M)
                                If yes → replace binary with adduser.exe
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe  → compile payload on Kali
move <binary> backup.exe      → backup original
move adduser.exe <binary>     → place payload
shutdown /r /t 0              → reboot if can't restart service manually
Get-LocalGroupMember Administrators → verify new user created
```

---

## DLL Hijacking — The Key Insight

DLL hijacking is not about exploiting a vulnerability in the DLL — it's about exploiting **Windows DLL search order**. Windows searches the application's own directory *before* System32. If:
1. The app tries to load a DLL that doesn't exist in its own directory
2. You can write files to that directory
3. The app runs at a higher privilege than you

→ Plant a malicious DLL with the correct name → it loads when the app starts → your code runs as the app's user.

Use Process Monitor to confirm which DLLs are missing: filter on `NAME NOT FOUND` + `Path contains .dll`.

---

## SeImpersonatePrivilege — The Fast Track to SYSTEM

If `whoami /priv` shows `SeImpersonatePrivilege` (Enabled), you can escalate to SYSTEM almost immediately using Potato attacks. This privilege is granted to service accounts by default (IIS, MSSQL, etc.).

```powershell
whoami /priv | findstr "Impersonate"
# SeImpersonatePrivilege  Impersonate a client after authentication  Enabled
```

→ Use **PrintSpoofer**, **GodPotato**, or **JuicyPotato** depending on Windows version.

---

## Connections

- [[PWK-Module-17-Windows-PrivEsc]] — full module with all commands, examples, and techniques
- [[PWK-Module-18-Linux-PrivEsc]] — Linux equivalent; compare decision trees
- [[Exploit-Discovery-Workflow]] — kernel exploit selection feeds into this tree
- The credential chain (find one → unlock next) is the most common OSCP exam path

---

*From: PWK Module 17*
