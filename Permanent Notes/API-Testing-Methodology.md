---
title: "API Testing Methodology — Black-Box Approach"
created: "2026-04-04"
type: permanent-note
status: active
tags:
  - permanent-note
  - api
  - web-attacks
  - methodology
  - oscp
---

# API Testing Methodology — Black-Box Approach

**Core idea:** REST APIs follow predictable patterns. Once you find one endpoint, you can systematically enumerate the rest. HTTP status codes reveal more than you'd expect — even "errors" tell you that an endpoint exists.

---

## The API Enumeration Mental Model

```
Discovery (what's there?) → Probe (how does it behave?) → Abuse (what can I do?)
```

**Step 1 — Discovery with Gobuster**
```bash
echo "{GOBUSTER}/v1" > pattern.txt
gobuster dir -u http://<target>:PORT -w /usr/share/wordlists/dirb/big.txt -p pattern.txt
```

**Step 2 — Probe with curl**
```bash
# Walk the path structure
curl -i http://<target>/api/v1            # list endpoints
curl -i http://<target>/api/v1/users      # list users → find admin
curl -i http://<target>/api/v1/users/admin  # probe specific user
```

**Step 3 — Decode HTTP status codes**

| Status | Meaning | Action |
|--------|---------|--------|
| `200 OK` | Success | Read the response carefully |
| `404 Not Found` + error message | Endpoint exists, record missing | Try different records |
| `405 Method Not Allowed` | Endpoint exists, wrong verb | Try GET/POST/PUT/DELETE/PATCH |
| `401 Unauthorized` | Auth required | Find auth endpoint, get token |
| `403 Forbidden` | Auth works, no permission | Look for privilege escalation |

**Step 4 — Abuse mass assignment**
```bash
# Send extra fields the API shouldn't accept
curl -d '{"username":"test","password":"test","admin":"True"}' \
  -H 'Content-Type: application/json' \
  http://<target>/api/v1/register

# If no error → mass assignment vulnerability confirmed
```

---

## Why "Error" Responses Are Informative

The key insight from PWK Module 08: a `404` with the message "User not found" is different from a `404` with "Endpoint not found." The first tells you the **endpoint exists** but you have the wrong parameter. Always read error messages — they leak information.

---

## Connections

- [[PWK-Module-08-Web-App-Attacks]] — full module notes with curl commands
- [[Burp-Suite-Workflow]] — use Repeater + Site map to track API testing
- OWASP API Security Top 10: BOLA, BFLA, Mass Assignment are all related
- Next: SQL injection can also affect APIs when they hit a database backend

---

*From: PWK Module 08, pages 225-231*
