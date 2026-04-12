---
title: "AWS SAA — Networking and CDN"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Networking and CDN

## Amazon VPC (Virtual Private Cloud)

### Overview
- Logically isolated section of the AWS cloud — your own virtual data center
- **Regional service** — one default VPC per region
- Default VPC: all subnets have internet access; instances get public + private IPs
- Custom VPCs: private by default; subnets and IGW must be created manually
- Up to **5 VPCs per region** (soft limit)
- Supports IPv4 and IPv6 (dual-stack)

### VPC Components Created with Custom VPC
When you create a custom VPC, these are automatically created:
- Route table (main)
- NACL (default)
- Security group (default)

**Not created automatically** (must create manually):
- Subnets
- Internet Gateway (and must attach it)

### CIDR and IP Addressing
- Assign IPv4 CIDR block at VPC creation (e.g., 10.0.0.0/16)
- Default VPC always uses **/16** CIDR
- Subnet CIDR: **/16** is largest, **/28** is smallest
- **/32** = single IP; **/0** = entire network
- AWS **reserves 5 IP addresses** per subnet (first 4 + last 1)
- Private IPs: communication within VPC; Public IPs: internet-accessible instances

### Subnets
- **Public subnet** — has route to Internet Gateway
- **Private subnet** — no direct internet route
- Subnets span a single AZ; cannot stretch across AZs
- New subnets automatically associated with main route table

> **Exam Tip:** Best practice — keep main route table private (no IGW route). Create a custom public route table for public subnets. New subnets inherit the main route table.

### Internet Gateway (IGW)
- Allows VPC instances with public IPs to access the internet
- One IGW per VPC maximum
- Performs NAT translation: maps private IPs → public IPs for outbound traffic
- Instances only know their private IP; IGW maintains the public IP mapping
- Initially detached when created — must manually attach to VPC

### Route Tables
- Every subnet associated with exactly one route table
- Route table controls where traffic is directed
- Can have multiple route tables; new subnets default to main route table
- Can configure routes to VPC Endpoints, VPN gateways, etc.

### NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|-------------|
| Managed | AWS-managed | Self-managed EC2 |
| HA | Built-in (within AZ) | Must configure manually |
| Bandwidth | Up to 45 Gbps | Depends on instance type |
| Patching | AWS handles | You handle |
| Status | **Preferred** | Deprecated (still works) |
| Cost | Per hour + data | EC2 instance cost |

- Both live in **public subnet**
- NAT Instance: must **disable Source/Destination Check** (it proxies traffic)
- For zone-independent HA: create a NAT Gateway per AZ, route each AZ's private traffic to its local NAT GW

> **Exam Tip:** NAT Gateway is always the preferred answer. NAT Instances are legacy/deprecated. For cost optimization, use NAT Gateway per AZ.

### Security Groups vs NACLs

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance level | Subnet level |
| State | **Stateful** (return traffic automatic) | **Stateless** (must allow return traffic explicitly) |
| Rules | Allow only | Allow AND deny |
| Rule evaluation | All rules evaluated | Rules evaluated in order (lowest number first) |
| Default | Deny all inbound; allow all outbound | Default NACL allows all; custom NACL denies all |
| Scope | Single VPC | Single VPC |

- Security Groups cannot span VPCs
- You can attach up to 5 security groups to an EC2 instance
- **Block malicious IPs** → use NACLs (not Security Groups — no deny rules)
- NACLs are evaluated **before** Security Groups
- Each subnet = exactly one NACL; one NACL can describe rules for many subnets

> **Exam Tip:** NACLs are stateless — you must explicitly allow BOTH inbound AND outbound traffic. If you add an inbound rule, you also need an outbound rule for the response. Security Groups are stateful — one rule covers both directions.

### VPC Peering
- Direct network connection between two VPCs using private IPs
- Can peer: same account, different account, same region, or **different regions**
- Instances behave as if on same network
- **No transitive peering** — A↔B and B↔C does NOT mean A↔C; must create A↔C directly
- **No overlapping CIDR blocks** allowed
- Not supported: overlapping CIDRs, transitive peering, edge-to-edge routing through gateway

### Transit Gateway
- **Hub-and-spoke** architecture for connecting multiple VPCs and on-premises networks
- Replaces complex VPC peering mesh
- Centrally managed; supports transitive routing (unlike VPC Peering)
- Can connect: VPCs, Direct Connect, Site-to-Site VPN, other AWS accounts
- Can share via RAM (Resource Access Manager) across accounts
- Supports route tables for traffic segregation

> **Exam Tip:** If you need to connect many VPCs (5+) or need transitive routing, use Transit Gateway instead of VPC Peering.

### VPC Endpoints
- Connect VPC to AWS services **without internet gateway, NAT, VPN, or Direct Connect**
- Traffic stays within AWS network; more secure and lower latency

| Type | Supports | Cost | How It Works |
|------|----------|------|--------------|
| **Gateway Endpoint** | S3, DynamoDB | Free | Entry in route table |
| **Interface Endpoint (PrivateLink)** | Most AWS services | $0.01/hr | ENI with private IP |

- Gateway Endpoint = route table entry (just a target, not a resource)
- Interface Endpoint = actual ENI in your VPC; uses DNS to route to private IP
- Secure Gateway Endpoint with VPC Endpoint Policies; Interface Endpoint with Security Groups

### VPC Flow Logs
- Captures IP traffic metadata (not packet contents) for network interfaces, subnets, VPCs
- Captures: source IP, destination IP, protocol, port, packet size, action (ACCEPT/REJECT)
- Sent to: **S3 bucket** or **CloudWatch Logs**
- Cannot capture: DHCP traffic, DNS to AWS servers, instance metadata queries
- Once created, **cannot change the configuration** — must create new log
- Can log valid traffic, invalid traffic, or both

### Bastion Host (Jump Box)
- Small EC2 instance in **public subnet**
- Used to SSH/RDP into instances in private subnet
- Best practice: restrict to single IP address or CIDR
- Pre-baked AMIs available
- Systems Manager Session Manager is a better alternative (no bastion, no open ports)

### AWS PrivateLink
- Secure private connectivity from your VPC to other VPCs or AWS services
- Does NOT traverse the internet
- For VPC-to-VPC within AWS: PrivateLink. For on-prem to AWS: Direct Connect.
- Publish your own endpoint for others to connect to from their VPCs

---

## Amazon Route 53

### Overview
- Highly available, scalable DNS web service
- Supports: domain registration, DNS routing, health checks
- AWS's own domain registrar

### DNS Record Types
- **A record** — URL → IPv4
- **AAAA record** — URL → IPv6
- **CNAME** — URL → another URL (not allowed for apex/naked domain)
- **Alias record** — URL → AWS resource (ELB, CloudFront, S3, etc.); can be used at apex domain
- **PTR record** — IPv4 → URL (reverse DNS lookup)
- **MX, TXT, SRV, NS, SOA, CAA** — other standard types

> **Exam Tip:** Alias records are preferred over CNAME for AWS resources (dynamic IPs, can be used at apex domain, free queries). CNAMEs cannot point to apex domain (zone root).

### Routing Policies

| Policy | Description | Key Behavior |
|--------|-------------|-------------|
| **Simple** | One record, one or more IPs | Returns random IP if multiple values |
| **Weighted** | Split traffic by percentage | Blue/green deployments, A/B testing |
| **Latency-based** | Route to lowest-latency region | Must create records per region |
| **Failover** | Active-passive failover | Monitor health of primary; switch to secondary |
| **Geolocation** | Route by user's location | Different content per country/continent |
| **Geoproximity** | Route by location + bias | Traffic flow, expand/shrink geographic coverage |
| **Multi-value** | Like Simple but with health checks | Only healthy IPs returned randomly |

- **Geo-proximity** requires Route53 Traffic Flow to be enabled; can use bias to shift traffic
- **Multivalue ≠ load balancer** — it returns multiple healthy IPs (client picks one)
- Health checks can monitor any internet-accessible endpoint

### DNSSEC
- Route 53 supports DNSSEC for enhanced security
- Protects against DNS spoofing and cache poisoning

### Route 53 Resolver
- Resolves DNS queries within VPCs
- Resolver Endpoints: inbound (on-prem → VPC), outbound (VPC → on-prem)

---

## Elastic Load Balancing (ELB)

### Load Balancer Types

| Type | Layer | Protocol | Key Features |
|------|-------|----------|--------------|
| **ALB** (Application) | Layer 7 | HTTP/HTTPS/gRPC | Path/host routing, WebSockets, Lambda targets |
| **NLB** (Network) | Layer 4 | TCP/UDP/TLS | Millions req/sec, static IP, ultra-low latency |
| **GWLB** (Gateway) | Layer 3+4 | IP | Deploy 3rd-party virtual appliances (firewalls, IDS/IPS) |
| **CLB** (Classic) | Layer 4+7 | HTTP/HTTPS/TCP | Legacy; not recommended for new apps |

### ALB (Application Load Balancer)
- Intelligent Layer 7 routing: path-based, host-based, header-based, query string
- **Listener rules** — route to different target groups based on conditions
- Target types: EC2 instances, IP addresses, Lambda functions, containers
- **Sticky sessions** — bind user to specific instance (via cookie); can defeat purpose of LB if overused
- Supports WebSockets and HTTP/2
- Cannot assign static IP — must use DNS name

### NLB (Network Load Balancer)
- Millions of requests/second with **ultra-low latency**
- **Can have static IP** (or Elastic IP) per AZ — useful when clients need IP whitelisting
- TLS termination supported
- Preserves source IP address of client
- Targets: EC2, IP addresses

### GWLB (Gateway Load Balancer)
- Deploy and scale third-party virtual appliances (firewalls, deep packet inspection, IDS/IPS)
- Single entry/exit point for all traffic; transparent network gateway
- Uses GENEVE protocol on port 6081

### CLB (Classic Load Balancer) — Legacy
- Supports HTTP/HTTPS or TCP (not both simultaneously)
- Supports sticky sessions and X-Forwarded-For headers
- Does **NOT support SNI** — use ALB or CloudFront instead

### ELB Cross-Zone Load Balancing
- Enabled: traffic distributed evenly across all instances in all AZs
- Disabled: each AZ node distributes only to its own AZ instances
- ALB: cross-zone enabled by default (free)
- NLB/GWLB: disabled by default (charges apply when enabled)

### ELB SSL/TLS
- SSL termination at LB — EC2 instances receive unencrypted traffic (less CPU load on instances)
- Supports **Perfect Forward Secrecy**
- **SNI (Server Name Indication)** — ALB and NLB support hosting multiple TLS certificates on one LB; CLB does NOT

> **Exam Tip:** ALB = intelligent HTTP routing, Lambda targets, WebSockets. NLB = extreme performance, static IP, Layer 4. CLB is legacy — avoid for new architectures. SNI not supported on CLB.

### ELB Health Checks
- Instances marked `InService` (healthy) or `OutOfService` (unhealthy)
- 504 error = application not responding behind LB (not LB fault itself)
- ELBs use DNS names, not predefined IPs (except NLB with static IP)

### ELB Advanced
- **X-Forwarded-For header** — original client IP forwarded to backend (via Proxy Protocol)
- **Path Patterns** — route different URL paths to different target groups (e.g., /api → API servers, /images → image servers)
- Load balancers are **regional** — do not balance across regions; provision one per region

---

## Amazon CloudFront

### Overview
- AWS CDN (Content Delivery Network) service
- Serves cached content from **edge locations** (globally distributed)
- Main components: **Origin** (S3, EC2, ELB, Route 53), **Edge Locations** (cache endpoints), **Distribution** (CDN network)
- Supports: dynamic, static, streaming, and interactive content

### CloudFront Key Details
- **TTL** (Time to Live) — how long content is cached at edge location (always in seconds)
- Edge locations are NOT read-only — can write to them (write returns to origin)
- Can manually **invalidate** cached content (incurs cost)
- Supports WebSocket, HTTP/HTTPS
- Origin failover: set up origin group with primary and secondary; automatic failover

### Origin Access Control (OAC) / Origin Access Identity (OAI)
- **OAI** (legacy) — virtual user; used to give CloudFront permission to access private S3 objects
- **OAC** (current/preferred) — more secure replacement for OAI; supports all S3 operations including SSE-KMS
- With OAC: S3 bucket policy allows only CloudFront to access; bucket stays private

### CloudFront Signed URLs vs Signed Cookies

| | Signed URLs | Signed Cookies |
|-|------------|----------------|
| Controls access to | Individual files | Multiple files / entire area |
| Use when | Single file restriction; client doesn't support cookies; RTMP | Multiple restricted files; don't want to change URLs |

> **Exam Tip:** Signed URLs = one file at a time. Signed Cookies = access to a group of files (e.g., premium content area). S3 pre-signed URLs are different — they grant time-limited direct access to S3 objects without CloudFront.

### CloudFront Geo-Restriction
- Whitelist: allow specific countries only
- Blacklist: block specific countries

### CloudFront vs Global Accelerator
- **CloudFront** — caches static content at edge, reduces origin load; content is cached
- **Global Accelerator** — routes traffic to origin via AWS backbone; no caching; improves routing for dynamic content
- Both use AWS edge locations, but different purposes

### Lambda@Edge vs CloudFront Functions

| Feature | Lambda@Edge | CloudFront Functions |
|---------|-------------|---------------------|
| Languages | Node.js, Python | JavaScript only |
| Max execution time | 5 sec (viewer), 30 sec (origin) | < 1 ms |
| Memory | 128MB – 10GB | 2MB max |
| Network access | Yes | No |
| Cost | Higher | Lower |
| Use case | Complex transformations, external API calls | Simple header rewrites, URL normalization |

---

## AWS Direct Connect

### Overview
- **Dedicated private network connection** between your premises and AWS
- Goes through a physical connection — does NOT traverse the internet
- Use cases: high throughput workloads, consistent/stable connection needed, regulatory requirements

### Connection Types
- **Dedicated Connections** — physical port dedicated to you; 1 Gbps, 10 Gbps, 100 Gbps
- **Hosted Connections** — provisioned by AWS Direct Connect Partner; pre-defined bandwidth

### Virtual Interfaces (VIFs)
- **Private VIF** — access VPC resources using private IPs
- **Public VIF** — access AWS public services (S3, DynamoDB) via Direct Connect
- **Transit VIF** — connect to Transit Gateway

### Direct Connect Gateway
- Connect multiple VPCs across regions or accounts via single Direct Connect connection

> **Exam Tip:** Direct Connect = dedicated private connection (no internet). VPN = encrypted connection over public internet. Direct Connect has lower latency and more consistent performance. Combine both for HA.

---

## AWS VPN

### Site-to-Site VPN
- Encrypted IPsec connection between on-premises network and AWS VPC
- Goes over the **public internet** (but encrypted)
- Requires: Virtual Private Gateway (AWS side), Customer Gateway (on-prem side)
- Quick to set up; good backup for Direct Connect
- Lower throughput than Direct Connect; internet-dependent

### AWS Client VPN
- Managed client-based VPN for accessing AWS and on-premises networks
- Uses OpenVPN protocol
- Users connect from any device using OpenVPN client

### VPN vs Direct Connect Summary
- **VPN**: internet, fast setup, lower cost, encrypted, variable latency
- **Direct Connect**: dedicated private line, consistent latency, higher throughput, higher cost

---

## AWS Global Accelerator

- Accelerates connectivity to improve performance and availability
- Uses **AWS global network** (backbone) — reduces number of hops
- Provides **2 static Anycast IP addresses** that route to nearest edge location
- Then traffic stays on AWS backbone to the destination
- Good for: non-cacheable dynamic content, improving global reach, reducing latency for non-HTTP traffic
- Provides fast regional failover

> **Exam Tip:** CloudFront = caching CDN for static content. Global Accelerator = routing optimization for both static and dynamic content (no caching). Route 53 latency routing = DNS-level only, no network optimization.

---

## Network Architecture Summary

| Problem | Solution |
|---------|---------|
| Private instances need internet access | NAT Gateway (public subnet) |
| Securely access S3/DynamoDB from VPC | VPC Gateway Endpoint (free) |
| Securely access other AWS services from VPC | VPC Interface Endpoint (PrivateLink) |
| Connect two VPCs same/different region | VPC Peering |
| Connect many VPCs + on-prem (hub-and-spoke) | Transit Gateway |
| Connect on-prem to AWS (private, high throughput) | Direct Connect |
| Connect on-prem to AWS (quick, over internet) | Site-to-Site VPN |
| Distribute global HTTP traffic | ALB + Route 53 |
| Extreme performance, millions req/sec | NLB |
| Cache static content globally | CloudFront |
| Optimize dynamic content routing globally | Global Accelerator |
| Block IPs/countries at network level | NACLs |
| Block web attacks (Layer 7) | WAF |

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — Security Groups, NACLs, WAF, VPC Flow Logs
- [[AWS-SAA-02-Compute]] — EC2 placement in subnets, ELB for Auto Scaling
- [[AWS-SAA-03-Storage]] — S3 with CloudFront OAC, Transfer Acceleration
- [[AWS-SAA-04-Databases]] — RDS in private subnet, ElastiCache in VPC
- [[AWS-SAA-06-Messaging-and-Integration]] — API Gateway, SQS/SNS in VPC
- [[AWS-SAA-07-Monitoring-and-Management]] — VPC Flow Logs, CloudTrail
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — 3-tier architecture, HA design patterns

*AWS SAA-C03/C04 Reference*
