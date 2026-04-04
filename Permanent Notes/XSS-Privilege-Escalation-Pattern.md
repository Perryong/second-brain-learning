---
title: "XSS Privilege Escalation — Admin Account Creation Pattern"
created: "2026-04-04"
type: permanent-note
status: active
tags:
  - permanent-note
  - xss
  - privilege-escalation
  - web-attacks
  - oscp
---

# XSS Privilege Escalation — Admin Account Creation Pattern

**Core idea:** When you have stored XSS in an admin-viewed page, you don't need to steal cookies — you can make the admin's browser perform admin actions *for* you.

---

## Why This Works

When an admin loads a page with your stored XSS payload, your JavaScript executes **in the context of their authenticated session**. The browser sends their cookies automatically on every request your JS makes — so your `XMLHttpRequest` to `/wp-admin/user-new.php` is treated as if the admin made it.

You bypass HttpOnly cookie protection entirely — because you're not stealing the cookie, you're *riding* the session.

---

## The Attack Pattern (4 Steps)

```
1. IDENTIFY   → Find stored XSS in an admin-visible location
                (comment section, log viewer, plugin dashboard, user profile)

2. CSRF TOKEN → Fetch the nonce/CSRF token from an admin page via XHR
                (can't submit forms without it — WordPress protects against CSRF)

3. ACTION     → Use the token to POST an admin action
                (create user, change password, install plugin)

4. DELIVER    → Encode the payload to bypass filters, inject via vector
                (User-Agent, comment field, profile field, etc.)
```

---

## When to Use This

- HttpOnly is set on session cookies (can't steal with `document.cookie`)
- Admin regularly visits a page you can inject into
- The app has no Content Security Policy (CSP) blocking inline scripts

---

## Key Technique: Encoding to Bypass Filters

```javascript
// Convert your JS to char codes in browser console:
function encode_to_javascript(string) {
    var output = '';
    for (pos = 0; pos < string.length; pos++) {
        output += string.charCodeAt(pos);
        if (pos != (string.length - 1)) output += ",";
    }
    return output;
}

// Deliver as:
eval(String.fromCharCode(118,97,114,...))
```

Useful when WAF blocks `<script>` tags but allows encoded variants.

---

## Connections

- Stored XSS is what makes this possible — see [[PWK-Module-08-Web-App-Attacks]]
- This escalates to RCE via WordPress plugin upload — next step after admin access
- CSRF protection (nonces) doesn't stop stored XSS because the JS runs *as the admin*
- DOM XSS follows the same pattern but the payload never touches the server

---

*From: PWK Module 08, pages 238-244*
