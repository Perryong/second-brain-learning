---
title: "PG Play: Sunset midnight"
created: "2023-07-28"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "Sunset midnight"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - suid-exploitation
  - cron-job-abuse
  - reverse-shell
  - file-upload
  - brute-force-hydra
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[Password-Cracking-Methodology]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: Sunset midnight

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-07-28

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 80/tcp (http)
- 3306/tcp (mysql)

**Key Techniques:**
- SUID Exploitation
- Cron Job Abuse
- Reverse Shell
- File Upload
- Brute Force (Hydra)
- WordPress

**Privilege Escalation Path:**
- Cron job abuse

---

## Full Walkthrough

Proving grounds Play - SunsetMidnight CTF writeup.

## NMAP
```sh
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Did not follow redirect to http://sunset-midnight/
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
3306/tcp open  mysql   MySQL 5.5.5-10.3.22-MariaDB-0+deb10u1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.22-MariaDB-0+deb10u1
```

## Fuzzing
## Files
```sh
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://sunset-midnight/FUZZ
Total requests: 37050

=====================================================================
ID           Response   Lines    Word       Chars       Payload                     
=====================================================================

000000005:   405        0 L      6 W        42 Ch       "xmlrpc.php"                
000000036:   200        87 L     300 W      4869 Ch     "wp-login.php"              
000000130:   200        97 L     823 W      7278 Ch     "readme.html"               
000000206:   200        384 L    3177 W     19915 Ch    "license.txt"               
000000248:   200        3 L      6 W        67 Ch       "robots.txt"                
000000263:   200        0 L      0 W        0 Ch        "wp-config.php"                            
000000413:   200        0 L      0 W        0 Ch        "wp-cron.php"                             
000000462:   200        11 L     24 W       228 Ch      "wp-links-opml.php"                           
000000838:   200        0 L      0 W        0 Ch        "wp-load.php"               
```

## Brute Forcing Mysql Credentials


## Logging into Mysql DB


Get user credentials.


Generate new password MD5  hash.


Update user password.


Sucessfuly logged into wordpress admin portal.

## Uploading Reverse Shell in themes

Uploading revershell in the themes resulted in failure.


### Generate Malicious wordpress plugin

[GitHub](https://github.com/wetw0rk/malicious-wordpress-plugin)

The python code allows to create malicious reverse shell payload and write it to the zip file.


Upload and install the malicious plugin


### Trigger reverse shell


Shell obtained



Post obtaining shell, hardcoded user credentials were found in the wordpress config files.

Found credentials for user `jose`


**SSH to user jose**


## Privilege Escalation

### SUIDs


The status binary in the SUID runs services.

- Create a service
- Apply executable permission
- Run `/usr/bin/status` binary

**Service file contents**

```sh
/bin/sh
```


**Root obtained**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[Password-Cracking-Methodology]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
