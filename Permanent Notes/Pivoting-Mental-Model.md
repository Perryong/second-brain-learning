---
title: "Pivoting — Mental Model and Technique Selection"
created: "2026-04-05"
type: permanent-note
status: active
tags:
  - permanent-note
  - pivoting
  - tunneling
  - ssh
  - methodology
  - oscp
---

# Pivoting — Mental Model and Technique Selection

**Core idea:** Pivoting is about using a compromised host as a relay to reach networks you can't touch directly. The technique you choose is determined by one question: **which direction can traffic flow?** If the target can reach you (outbound), use forward tunneling. If it can't (inbound blocked), use reverse tunneling.

---

## The One-Question Decision Rule

```
Can the pivot host initiate connections OUT to your Kali machine?
├── YES → Use SSH -L (local forward) or SSH -D (SOCKS proxy)
└── NO  → Use SSH -R (remote forward) — pivot calls home, you work backwards
```

Every pivoting decision flows from this single question about firewall direction.

---

## The Four Techniques at a Glance

| Technique | Who initiates SSH | Where SOCKS/listener binds | Firewall requirement |
|-----------|-----------------|---------------------------|----------------------|
| Socat port forward | — (not SSH) | CONFLUENCESERVER:8090 | Outbound from CONF → internal |
| SSH -L local forward | Kali → pivot | Kali's port → pivot → target | Kali can reach pivot |
| SSH -D dynamic SOCKS | Kali → pivot | SOCKS on Kali (127.0.0.1:1080) | Kali can reach pivot |
| SSH -R remote forward | Pivot → Kali | Kali-side listener via pivot | Pivot can reach Kali outbound |

**The trap:** SSH -L and -D both require Kali to initiate the connection. If the pivot is behind a strict inbound firewall, these won't work — you need -R.

---

## Why `0.0.0.0` Matters for SSH -L

By default, SSH -L binds `127.0.0.1` — only loopback, only your machine. If you have other hosts (like a different hop) that need to reach your forwarded port, bind `0.0.0.0` explicitly:

```bash
ssh -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215
```

This is the difference between "only I can use this tunnel" and "anyone who can reach my Kali machine can use this tunnel" — critical when chaining hops.

---

## Proxychains Constraints — The Hidden Cost of SOCKS

SSH -D gives you a SOCKS proxy, which sounds powerful. But Proxychains has real limitations:

- **No UDP** — eliminates most nmap scan types; must use `-sT` (full TCP connect)
- **No ICMP** — `ping` and host discovery are dead; must use `-Pn`
- **No DNS** — hostnames fail; must use `-n` and supply raw IPs
- **Slow** — SOCKS adds latency per connection; nmap through Proxychains is slow

**Rule:** When running nmap through Proxychains, always add `-sT -Pn -n`. Without these three flags, you'll get misleading results or no results at all.

---

## The Remote Tunnel Mental Model

SSH -R feels backwards at first. The pivot machine (your foothold) connects **outward to Kali** — so even if inbound connections to the pivot are blocked by a firewall, the session is established from the pivot's side. Once the tunnel is up, Kali gets a local port that forwards through the pivot to internal targets:

```
[Kali]<──SSH session initiated by pivot──[CONFLUENCE]──[Internal network]
  ↑
  Kali binds 127.0.0.1:2345
  psql -h 127.0.0.1 -p 2345 → goes through tunnel → reaches internal PostgreSQL
```

**The key insight:** You're not bypassing the firewall — you're exploiting the direction it allows. Firewalls that block inbound but allow outbound SSH are defeated by -R.

---

## Detecting the Multi-Homed Pivot

Before you can pivot, you need to know you're on a multi-homed host and what networks are reachable:

```bash
ip addr    # all interfaces and IPs
ip route   # routing table — shows which networks are directly reachable
```

Look for a second interface (e.g., `ens192`) with a different subnet — that's your pivot path into the internal network.

---

## The Bash Discovery Loop Pattern

When you can't bring tools to the pivot host, fall back to bash:

```bash
# Host sweep — who's alive?
for i in $(seq 1 254); do
  (ping -c 1 172.16.50.$i | grep "bytes from" &)
done

# Port sweep — what's open on a host?
for port in 22 80 443 445 3306 5432 8080 8443; do
  (echo >/dev/tcp/172.16.50.217/$port) 2>/dev/null && echo "$port open"
done
```

This works on any Linux host with bash — no nmap, no masscan, no special tools required.

---

## Connections

- [[PWK-Module-19-Port-Redirection-SSH-Tunneling]] — full module with all commands, topology diagrams, and Confluence scenario
- [[Windows-PrivEsc-Decision-Tree]] — once you pivot in, Windows PrivEsc determines next steps
- [[Linux-PrivEsc-Decision-Tree]] — same for Linux pivot targets
- [[Exploit-Discovery-Workflow]] — pivoting often exposes new services to exploit on internal hosts

---

*From: PWK Module 19*
