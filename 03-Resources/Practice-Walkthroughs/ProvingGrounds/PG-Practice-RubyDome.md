---
title: "PG Practice: RubyDome"
created: "2023-08-27"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Practice
machine: "RubyDome"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-practice
  - linux
  - ctf
  - oscp
related:
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Practice: RubyDome

> **Platform:** Proving Grounds Practice | **OS:** Linux | **Date:** 2023-08-27

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 3000/tcp (http)

**Key Techniques:**
- (See walkthrough)

**Privilege Escalation Path:**
- Kernel exploit

---

## Full Walkthrough

Proving grounds Practice - RubyDome CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 b9bc8f013f855df95cd9fbb615a01e74 (ECDSA)
|_  256 53d97f3d228afd5798fe6b1a4cac7967 (ED25519)
3000/tcp open  http    WEBrick httpd 1.7.0 (Ruby 3.0.2 (2021-07-07))
|_http-title: RubyDome HTML to PDF
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-server-header: WEBrick/1.7.0 (Ruby/3.0.2/2021-07-07)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web


WEBrick 1.7.0 is vulnerable to Command injection and as per the above screenshot the application gets the URL and converts the page content into pdf. Passing malicious inputs or invalid URL resulted in server error with exception shown in the web page.


The PDFKit is used to convert the contents to pdf. The PDFkit used in the application is vulnerable to [Command Injection](https://www.exploit-db.com/exploits/51293).


**Initial Foothold Obtained**

## Privilege Escalation

Check the system user executable permissions.


As per the above image the user `andrew` can run the file `app.rb` using ruby as sudo user without password.

Add the below content to the `app.rb` file and execute the file using `/usr/bin/ruby` as super user.

```sh
echo 'exec "/bin/bash"' > app.rb
```


**Root Obtained**

---

## Technique Links

- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
