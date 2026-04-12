---
title: "PG Practice: ClamAV"
created: "2023-08-20"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "ClamAV"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - linux
  - ctf
  - oscp
  - reverse-shell
  - smb-enumeration
  - metasploit
  - searchsploit--exploit-db
  - snmp-enumeration
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[SMB-Enumeration-Exploitation]]"
  - "[[Metasploit-Mental-Model]]"
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[Nmap-Enumeration-Methodology]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
---

# PG Practice: ClamAV

> **Platform:** Proving Grounds Practice | **OS:** Linux | **Date:** 2023-08-20

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 25/tcp (smtp)
- 80/tcp (http)
- 139/tcp (netbios-ssn)
- 199/tcp (smux)
- 445/tcp (netbios-ssn)
- 60000/tcp (ssh)

**Key Techniques:**
- Reverse Shell
- SMB Enumeration
- Metasploit
- Searchsploit / Exploit-DB
- SNMP Enumeration
- SMTP

**Privilege Escalation Path:**
- Kernel exploit

---

## Full Walkthrough

Proving grounds Practice - ClamAV CTF writeup.

## Nmap

```sh
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
| ssh-hostkey: 
|   1024 303ea4135f9a32c08e46eb26b35eee6d (DSA)
|_  1024 afa2493ed8f226124aa0b5ee6276b018 (RSA)
25/tcp    open  smtp        Sendmail 8.13.4/8.13.4/Debian-3sarge3
| smtp-commands: localhost.localdomain Hello [192.168.45.203], pleased to meet you, ENHANCEDSTATUSCODES, PIPELINING, EXPN, VERB, 8BITMIME, SIZE, DSN, ETRN, DELIVERBY, HELP
|_ 2.0.0 This is sendmail version 8.13.4 2.0.0 Topics: 2.0.0 HELO EHLO MAIL RCPT DATA 2.0.0 RSET NOOP QUIT HELP VRFY 2.0.0 EXPN VERB ETRN DSN AUTH 2.0.0 STARTTLS 2.0.0 For more info use "HELP <topic>". 2.0.0 To report bugs in the implementation send email to 2.0.0 sendmail-bugs@sendmail.org. 2.0.0 For local information send email to Postmaster at your site. 2.0.0 End of HELP info
80/tcp    open  http        Apache httpd 1.3.33 ((Debian GNU/Linux))
|_http-server-header: Apache/1.3.33 (Debian GNU/Linux)
|_http-title: Ph33r
| http-methods: 
|   Supported Methods: GET HEAD OPTIONS TRACE
|_  Potentially risky methods: TRACE
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
199/tcp   open  smux        Linux SNMP multiplexer
445/tcp   open  netbios-ssn Samba smbd 3.0.14a-Debian (workgroup: WORKGROUP)
60000/tcp open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
| ssh-hostkey: 
|   1024 303ea4135f9a32c08e46eb26b35eee6d (DSA)
|_  1024 afa2493ed8f226124aa0b5ee6276b018 (RSA)
Service Info: Host: localhost.localdomain; OSs: Linux, Unix; CPE: cpe:/o:linux:linux_kernel
```

## Attack Vector

PORT: 25 => Sendmail 8.13.4/8.13.4/Debian-3sarge3

## SNMP Check


The service is running with `black-hole-mode` enabled which is vulnerable to remote code execution.

## Searcsploit Search

As shown in the above snmp recon the `black-hole-mode` is enabled on `clamav-milter` process.

```sh
naveenj@hackerspace:|00:49|~$ searchsploit Sendmail clamav
------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                       |  Path
------------------------------------------------------------------------------------- ---------------------------------
ClamAV Milter 0.92.2 - Blackhole-Mode (Sendmail) Code Execution (Metasploit)         | multiple/remote/9913.rb
Sendmail with clamav-milter < 0.91.2 - Remote Command Execution                      | multiple/remote/4761.pl
------------------------------------------------------------------------------------- ---------------------------------
```

## Exploitation

Run the perl script to obtain the remode code execution.
Add the following `#!/bin/perl` to the perl script before running it.


As shown above connect to the port mentioned using `nc` to obtain reverse shell.


Root obtained.

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[SMB-Enumeration-Exploitation]]
- [[Metasploit-Mental-Model]]
- [[Searchsploit-and-CVE-Workflow]]
- [[Nmap-Enumeration-Methodology]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Proving-Grounds-Master-Index]]
