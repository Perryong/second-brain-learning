---
title: "PG Practice: Algernon"
created: "2023-08-26"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Algernon"
os: Windows
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - windows
  - ctf
  - oscp
  - searchsploit--exploit-db
  - netcat
related:
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Algernon

> **Platform:** Proving Grounds Practice | **OS:** Windows | **Date:** 2023-08-26

---

## Quick Summary

**Open Ports:**
- 21/tcp (ftp)
- 80/tcp (http)
- 135/tcp (msrpc)
- 139/tcp (netbios-ssn)
- 445/tcp (microsoft-ds?)
- 5040/tcp (unknown)
- 9998/tcp (http)
- 17001/tcp (remoting)
- 49664/tcp (msrpc)
- 49665/tcp (msrpc)
- 49666/tcp (msrpc)
- 49667/tcp (msrpc)
- 49668/tcp (msrpc)
- 49669/tcp (msrpc)

**Key Techniques:**
- Searchsploit / Exploit-DB
- Netcat

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Practice - Algernon CTF writeup.

## Nmap

```sh
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd => Anonymous login
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
9998/tcp  open  http          Microsoft IIS httpd 10.0
17001/tcp open  remoting      MS .NET Remoting services
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
```

## Directory Fuzzing

```text
http://192.168.172.65/aspnet_client/
http://192.168.172.65:9998/interface/root#/login
```

## PORT: 9998


### Searchsploit


Change the IP addressess and PORT in the exploit code and run netcat listener on the PORT specified.


Run the python exploit.


**Shell Obtained**

---

## Technique Links

- [[Searchsploit-and-CVE-Workflow]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
