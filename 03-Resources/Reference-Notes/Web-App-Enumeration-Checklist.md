---
title: "Web Application Enumeration Checklist"
created: "2026-04-04"
type: reference-note
status: active
tags:
  - oscp
  - web-attacks
  - checklist
  - enumeration
  - reference
---

# Web Application Enumeration Checklist

> Use this every time you encounter a web application on a target. Run top-to-bottom before attempting exploitation.

---

## Phase 1 — Server Fingerprinting

```bash
# 1. Nmap service scan
sudo nmap -p80,443,8080,8443 -sV <ip>

# 2. Nmap NSE http-enum (directory + file discovery)
sudo nmap -p80 --script=http-enum <ip>

# 3. Check robots.txt and sitemap.xml
curl http://<ip>/robots.txt
curl http://<ip>/sitemap.xml
```

**Checklist:**
- [ ] Web server software + version identified
- [ ] OS identified (from Nmap or headers)
- [ ] robots.txt checked for disallowed paths
- [ ] sitemap.xml checked for all pages

---

## Phase 2 — Technology Stack Identification

- [ ] Wappalyzer lookup on the domain
- [ ] Check response headers: `Server`, `X-Powered-By`, `X-Aspnet-Version`, `x-amz-cf-id`
- [ ] Check page source for JS library versions (jQuery, Bootstrap, React)
- [ ] Check URL extensions: `.php`, `.asp`, `.aspx`, `.jsp`, `.do`
- [ ] Check Firefox Debugger for JavaScript framework files

```bash
# View response headers with curl
curl -I http://<ip>
```

---

## Phase 3 — Directory and File Discovery

```bash
# Small wordlist (fast, initial pass)
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirb/common.txt -t 10

# Larger wordlist (thorough)
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirb/big.txt -t 10

# With file extensions
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak,old -t 10
```

**Look for:**
- [ ] Login pages (`/login`, `/admin`, `/wp-login.php`, `/administrator`)
- [ ] Upload directories (`/uploads`, `/files`, `/attachments`)
- [ ] Backup files (`.bak`, `.old`, `.zip`, `~`)
- [ ] Config files (`.env`, `config.php`, `web.config`)
- [ ] API endpoints (`/api`, `/v1`, `/v2`)

---

## Phase 4 — Page Content Inspection

Open Firefox DevTools (F12) on every interesting page:

- [ ] **Inspector:** Hidden form fields (`<input type="hidden">`)
- [ ] **Debugger:** JS libraries and versions, source map files
- [ ] **Network:** Observe all requests made by the page (XHR, API calls)
- [ ] **Storage:** Cookie flags (HttpOnly, Secure), localStorage, sessionStorage
- [ ] **Console:** Any JS errors that reveal stack traces or paths

---

## Phase 5 — Input Field Mapping

For every input field found:

- [ ] What does it accept? (text, numbers, file, email)
- [ ] Where does the input appear in the response?
- [ ] Test with special chars: `< > ' " { } ;`
- [ ] Test with long input (overflow?)
- [ ] Is the parameter in the URL? (GET param) → try manipulating it

---

## Phase 6 — API Discovery and Testing

- [ ] Enumerate API endpoints with Gobuster + pattern file
- [ ] Test HTTP verbs on each endpoint (GET, POST, PUT, DELETE, PATCH)
- [ ] Look for mass assignment (submit extra fields like `admin=true`)
- [ ] Test authentication: what happens without a token? With expired token?
- [ ] Check for IDOR: change user ID in request to access other users' data

---

## Phase 7 — XSS Testing

Test every input that reflects back in the response:

- [ ] `<script>alert(1)</script>`
- [ ] `"><script>alert(1)</script>`
- [ ] `'><script>alert(1)</script>`
- [ ] `<img src=x onerror=alert(1)>`
- [ ] HTTP headers: User-Agent, Referer, X-Forwarded-For

---

## Tool Quick Reference

| Task | Tool | Command |
|------|------|---------|
| Port/service scan | nmap | `sudo nmap -p80 -sV <ip>` |
| Directory discovery | nmap | `sudo nmap -p80 --script=http-enum <ip>` |
| Directory brute force | gobuster | `gobuster dir -u http://<ip> -w common.txt -t 10` |
| Technology stack | wappalyzer | Browser extension or website |
| Intercept requests | burpsuite | Set Firefox proxy to 127.0.0.1:8080 |
| API testing | curl | `curl -i http://<ip>/api/v1/users` |
| Robots/sitemap | curl | `curl http://<ip>/robots.txt` |

---

## Links

- [[PWK-Module-08-Web-App-Attacks]] — full theory + examples
- [[PWK-Module-09-Common-Web-App-Attacks]] — Directory Traversal, File Inclusion, Command Injection
- [[OSCP-Certification-Sprint]] — parent project
