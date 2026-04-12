---
title: "PG Practice: Payday"
created: "2023-10-19"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Payday"
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
  - brute-force-hydra
  - brute-force-ncrack
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[SMB-Enumeration-Exploitation]]"
  - "[[Password-Cracking-Methodology]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Payday

> **Platform:** Proving Grounds Practice | **OS:** Linux | **Date:** 2023-10-19

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 80/tcp (http)
- 110/tcp (pop3)
- 139/tcp (netbios-ssn)
- 143/tcp (imap)
- 445/tcp (netbios-ssn)
- 993/tcp (ssl/imap)
- 995/tcp (ssl/pop3)
- 80/tcp (http)

**Key Techniques:**
- Reverse Shell
- SMB Enumeration
- Brute Force (Hydra)
- Brute Force (ncrack)

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Practice - Payday CTF writeup.

## Nmap

```text
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.6p1 Debian 5build1 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.4 ((Ubuntu) PHP/5.2.3-1ubuntu6)
110/tcp open  pop3        Dovecot pop3d
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: MSHOME)
143/tcp open  imap        Dovecot imapd
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: MSHOME)
993/tcp open  ssl/imap    Dovecot imapd
995/tcp open  ssl/pop3    Dovecot pop3d
```

## 80/tcp  open http Apache httpd 2.2.4 ((Ubuntu) PHP/5.2.3-1ubuntu6)

### Directory Search


**Admin login**

http://192.168.195.39/admin

> Credential: `admin:admin`

CS-cart software is vulnerable to [Remote Code Execution](https://www.exploit-db.com/exploits/48891).

Rename the pentest monkey php reverse shell to `.phtml` file and uplaod it to the template editor menu as new template. Direct the below url to trigger the reverse shell.

http://192.168.195.39/skins/shell.phtml


**Initial Foothold Obtained**

## Privilege Escalation

Upon recon it was found that there are 2 users in the system which are `patrick` and `root`.

Brute forcing the SSH credentials for the user patrick using hydra have failed and as an alternative used `ncrack` to crack the password.

> Credential: `patrick:patrick`


> Note: Always try to login to SSH using password as same as username.

Upon recon it was found that the user patrick can run ALL, run `sudo su` to obtain root.


**Root Obtained**

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[SMB-Enumeration-Exploitation]]
- [[Password-Cracking-Methodology]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
