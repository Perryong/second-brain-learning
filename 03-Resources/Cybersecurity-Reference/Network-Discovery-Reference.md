---
title: "Network Discovery Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - network
  - enumeration
  - discovery
  - nmap
  - scanning
---

# Network Discovery Reference

Quick reference for network discovery, host enumeration, and scanning techniques.

**Related Notes:**
- [[AD-Enumeration-Reference]]
- [[Network-Pivoting-Reference]]
- [[Nmap-Enumeration-Methodology]]

---

## DHCP

```bash
# Discover DHCP server and offered config
sudo nmap --script broadcast-dhcp-discover
```

---

## DNS

```bash
# AD DNS lookups
nslookup -type=srv _ldap._tcp.dc._msdcs.<domain>    # LDAP / DC
nslookup -type=srv _kerberos._tcp.<domain>           # KDC
nslookup -type=srv _ldap._tcp.<domain>               # Global Catalog
```

---

## NBT-NS / NetBIOS

```bash
nbtscan -r 192.168.1.0/24       # Scan range for NetBIOS names
nmblookup -A <IP>               # Get name for a single IP
```

---

## MDNS (Zero-conf)

```bash
mdns-scan
```

---

## ARP

```bash
# View ARP neighbors (Linux)
ip neigh

# ARP scan with nmap (requires root)
sudo nmap -sn -n 192.168.122.0/24

# ARP scan with arp-scan
sudo arp-scan -l

# ARP spoof with arpspoof
arpspoof -i wlan0 -t 10.0.0.X 10.0.0.Y

# ARP spoof with Bettercap
sudo bettercap -iface wlan0
net.probe on
set arp.spoof.targets <target_IP>
arp.spoof on
net.sniff on
```

---

## Ping Sweep

```bash
# Nmap ping sweep (no port scan, no DNS)
nmap -sn -n --disable-arp-ping 192.168.1.1-254 | grep -v "host down"

# Bash one-liner
for i in `seq 1 255`; do ping -c 1 -w 1 192.168.1.$i > /dev/null 2>&1; if [ $? -eq 0 ]; then echo "192.168.1.$i is UP"; fi; done
```

---

## LDAP

```bash
# Null bind connection
ldapsearch -x -h <ip> -s base
```

---

## Nmap

### Basic Scans

```bash
# Full port scan with version detection
sudo nmap -sSV -p- 192.168.0.1 -oA OUTPUTFILE -T4

# From input file
sudo nmap -sSV -oA OUTPUTFILE -T4 -iL INPUTFILE.csv

# CTF quick scan
nmap -sV -sC -oA ~/nmap-initial 192.168.1.1

# Aggressive scan (OS detection, version, scripts, traceroute)
nmap -A -T4 <target>
```

### Nmap Scripts

```bash
# Default scripts
nmap -sC <target>

# HTTP enumeration
nmap --script 'http-enum' -v <target> -p80

# SMB user enumeration
nmap --script smb-enum-users.nse -p 445 <target>

# Combine with searchsploit
nmap -p- -sV -oX a.xml <IP>; searchsploit --nmap a.xml

# HTML report
nmap -sV <IP> -oX scan.xml && xsltproc scan.xml -o "$(date +%m%d%y)_report.html"

# List all scripts
ls /usr/share/nmap/scripts/
```

---

## Masscan

```bash
# Full port scan
masscan -iL ips-online.txt --rate 10000 -p1-65535 --only-open -oL masscan.out
masscan -e tun0 -p1-65535,U:1-65535 10.10.10.97 --rate 1000

# Find machines on network
sudo masscan --rate 500 --interface tap0 --router-ip $ROUTER_IP --top-ports 100 $NETWORK -oL masscan_machines.tmp
cat masscan_machines.tmp | grep open | cut -d " " -f4 | sort -u > machines.lst

# TCP/UDP banner grab after masscan
TCP_PORTS=$(cat masscan.lst | grep open | grep tcp | cut -d " " -f3 | tr '\n' ',' | head -c -1)
[ "$TCP_PORTS" ] && sudo nmap -sT -sC -sV -v -Pn -n -T4 -p$TCP_PORTS --reason -oA nmap_tcp $MACHINE_IP

UDP_PORTS=$(cat masscan.lst | grep open | grep udp | cut -d " " -f3 | tr '\n' ',' | head -c -1)
[ "$UDP_PORTS" ] && sudo nmap -sU -sC -sV -v -Pn -n -T4 -p$UDP_PORTS --reason -oA nmap_udp $MACHINE_IP
```

---

## Port Check Without Nmap

```bash
# Check if specific ports are open on a host
for i in {21,22,80,139,443,445,3306,3389,8080,8443}; do nc -z -w 1 192.168.1.18 $i > /dev/null 2>&1; if [ $? -eq 0 ]; then echo "port $i open"; fi; done

# Combined: ping sweep + port check on /24
for i in `seq 1 255`; do
    ping -c 1 -w 1 192.168.1.$i > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo "192.168.1.$i is UP:"
        for j in {21,22,80,139,443,445,3306,3389,8080,8443}; do
            nc -z -w 1 192.168.1.$i $j > /dev/null 2>&1
            if [ $? -eq 0 ]; then echo "  port $j open"; fi
        done
    fi
done
```

---

## Netdiscover

```bash
netdiscover -i eth0 -r 192.168.1.0/24
```

---

## Responder (Passive Listening)

```bash
responder -I eth0 -A       # Listen only — don't respond (safe recon)
responder.py -I eth0 -wrf  # Full poisoning mode
```

---

## MITM Techniques

```bash
# DHCP poisoning
responder --interface "eth0" --DHCP --wpad

# Bettercap
bettercap -X --proxy --proxy-https -T <target_IP>
```

### SSL MITM with OpenSSL

```bash
# 1. On client: redirect hostname to your MITM server
echo "<MITM_IP> domain.target.com" >> /etc/hosts

# 2. Generate self-signed cert on MITM server
openssl req -subj '/CN=domain.target.com' -batch -new -x509 -days 365 -nodes -out server.pem -keyout server.pem

# 3. Set up MITM pipe
mkfifo response
sudo openssl s_server -cert server.pem -accept <INTERFACE>:<PORT> -quiet < response | tee | \
  openssl s_client -quiet -servername domain.target.com -connect <SERVER_IP>:<PORT> | tee | cat > response
```
