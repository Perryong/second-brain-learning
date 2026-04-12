---
title: "PG Play: FunboxEasyEnum"
created: "2023-09-15"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "FunboxEasyEnum"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - gtfobins-exploit
  - sudo-misconfiguration
  - reverse-shell
  - file-upload
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: FunboxEasyEnum

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-15

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 80/tcp (http)

**Key Techniques:**
- GTFObins Exploit
- Sudo Misconfiguration
- Reverse Shell
- File Upload

**Privilege Escalation Path:**
- sudo -l abuse
- Kernel exploit

---

## Full Walkthrough

Proving grounds Play - FunboxEasyEnum CTF writeup.

## Nmap

```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 9c52325b8bf638c77fa1b704854954f3 (RSA)
|   256 d6135606153624ad655e7aa18ce564f4 (ECDSA)
|_  256 1ba9f35ad05183183a23ddc4a9be59f0 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web PORT: 80

```text
http://192.168.177.132/mini.php
```

## Upload reverse shell


Trigger reverse shell by visiting `192.168.177.132/shell.php` URL.

**Initial Foothold obtained**


## User Enumeration

```text
goat
harry
karla
oracle
sally
```

The above are the list of users who have home directory in the machine. The user `goat` has the password configured as same as username `goat`.

SSH to the user `goat` or switch user.

## Privilege Escalation

Enumerate executable permissions for the user goat.

```sh
t@funbox7:~$ sudo -l
Matching Defaults entries for goat on funbox7:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User goat may run the following commands on funbox7:
    (root) NOPASSWD: /usr/bin/mysql
```

Search for sudo exploit on GTFOBins for `mysql`.



**Root Obtained**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
