---
title: "AWS SAA — Well-Architected and Architecture Patterns"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Well-Architected and Architecture Patterns

## AWS Well-Architected Framework

### The 6 Pillars

| Pillar | Focus | Key Services |
|--------|-------|-------------|
| **Operational Excellence** | Run, monitor, and improve systems; automate operations | CloudFormation, CodePipeline, CloudWatch, Systems Manager |
| **Security** | Protect data, systems, assets; risk assessment | IAM, KMS, WAF, Shield, GuardDuty, CloudTrail, Config |
| **Reliability** | Recover from failures, acquire resources dynamically, mitigate disruptions | Multi-AZ, Auto Scaling, Route 53 health checks, S3 versioning |
| **Performance Efficiency** | Use resources efficiently; adapt as demand changes | Right-sizing, Serverless, CloudFront, ElastiCache, Aurora |
| **Cost Optimization** | Avoid unnecessary costs | Spot/Reserved instances, Savings Plans, Cost Explorer, S3 lifecycle |
| **Sustainability** | Minimize environmental impact | Right-sizing, Serverless, Managed services, Graviton instances |

### Operational Excellence Key Practices
- Infrastructure as Code (CloudFormation)
- Annotate documentation
- Make frequent, small, reversible changes
- Refine operations procedures frequently
- Anticipate failure
- Learn from all operational failures

### Security Key Practices
- Implement a strong identity foundation (least privilege)
- Enable traceability (CloudTrail, Config)
- Apply security at all layers (VPC, Security Groups, WAF, encryption)
- Automate security best practices
- Protect data in transit and at rest
- Prepare for security events (incident response)

### Reliability Key Practices
- Test recovery procedures
- Automatically recover from failure
- Scale horizontally to increase availability
- Stop guessing capacity (use Auto Scaling)
- Manage change in automation

### Performance Efficiency Key Practices
- Democratize advanced technologies (managed services)
- Go global in minutes (CloudFront, Global Accelerator)
- Use serverless architectures
- Experiment more often
- Consider mechanical sympathy (choose the right technology)

### Cost Optimization Key Practices
- Implement Cloud Financial Management
- Adopt a consumption model (pay for what you use)
- Measure overall efficiency (right-sizing)
- Stop spending on data center operations
- Analyze and attribute expenditure

### Sustainability Key Practices
- Understand your impact
- Establish sustainability goals
- Maximize utilization (shared services, right-sizing)
- Anticipate and adopt new, more efficient offerings
- Use managed services
- Reduce downstream impact (Graviton, serverless)

---

## High Availability (HA) vs Fault Tolerance

| Concept | Definition | Goal |
|---------|-----------|------|
| **High Availability** | System remains accessible even if components fail | Minimize downtime |
| **Fault Tolerance** | System continues operating without interruption during component failure | Zero downtime |
| **Disaster Recovery** | Recover from large-scale failures (region-level) | Minimize data loss and recovery time |

- HA allows brief downtime; Fault Tolerance allows none
- HA: Multi-AZ deployments, ELB, Auto Scaling
- Fault Tolerance: Multi-region with active-active, DynamoDB Global Tables, Aurora Global

---

## RTO vs RPO

| Term | Definition | Goal |
|------|-----------|------|
| **RTO** (Recovery Time Objective) | Maximum acceptable time to restore service after a disaster | Minimize downtime |
| **RPO** (Recovery Point Objective) | Maximum acceptable amount of data loss (measured in time) | Minimize data loss |

- Lower RTO/RPO = more expensive (requires more infrastructure)
- RTO = "How fast can we recover?" RPO = "How much data can we afford to lose?"

---

## Disaster Recovery Strategies

Listed from least → most expensive / most → least downtime:

### 1. Backup and Restore
- **RTO: Hours | RPO: Hours to days**
- Cheapest; backup data to S3/Glacier; restore from backups when disaster occurs
- No infrastructure running in DR region (minimal cost)
- Use for: non-critical systems, large data with acceptable RTO

### 2. Pilot Light
- **RTO: 10s of minutes | RPO: Minutes**
- Minimal version of environment always running in DR region
- Core components (DB replicated) active; other components off
- Scale up quickly in disaster by launching pre-built resources
- "Small flame that can quickly ignite the full furnace"

### 3. Warm Standby
- **RTO: Minutes | RPO: Seconds to minutes**
- Scaled-down but fully functional copy running in DR region
- Can handle some traffic at reduced capacity; scale up on failover
- Higher cost than pilot light but faster recovery

### 4. Multi-Site (Active-Active)
- **RTO: Seconds | RPO: Near zero**
- Full production environment in multiple regions
- Both regions serve traffic simultaneously
- Route 53 routes traffic to both; instant failover
- Most expensive; best for mission-critical systems

> **Exam Tip:** Order from cheapest/slowest to most expensive/fastest: Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active-Active. Match the scenario to the required RTO/RPO.

---

## Common Architecture Patterns

### 3-Tier Web Application Architecture

```
Internet → Route 53 → CloudFront (CDN)
           ↓
           ELB (ALB) in public subnets
           ↓
           EC2 / ECS (Auto Scaling Group) in private subnets (Web/App tier)
           ↓
           RDS Multi-AZ / Aurora in private subnets (DB tier)
           ↓
           ElastiCache for caching
```

- **Web tier**: ELB (public subnet), accepts internet traffic
- **App tier**: EC2/ECS (private subnet), business logic
- **DB tier**: RDS/Aurora (private subnet), data persistence
- **Caching**: ElastiCache between app and DB tiers

### Serverless Architecture

```
Client → API Gateway → Lambda → DynamoDB
                    ↓
                    S3 (static content served via CloudFront)
```

- No servers to manage
- Lambda for compute, DynamoDB for data, S3 for storage
- API Gateway as the frontend
- CloudFront for static content
- Good for: variable traffic, event-driven workloads, cost-sensitive apps

### Event-Driven Architecture

```
Event Source → EventBridge / SNS → Multiple Consumers
                                 → Lambda (process)
                                 → SQS (buffer)
                                 → Kinesis (stream)
                                 → Step Functions (workflow)
```

- Producers emit events; consumers react independently
- Loose coupling — producers don't know about consumers
- Scale independently; resilient to failures
- Good for: microservices, async processing, notifications

### CQRS (Command Query Responsibility Segregation)
- Separate read and write operations into different models
- **Write path**: commands → DynamoDB or RDS (write-optimized)
- **Read path**: queries → Elasticsearch, DynamoDB, ElastiCache (read-optimized)
- Use DynamoDB Streams to sync write → read models
- Benefits: optimize each side independently; scale reads without impacting writes

### Microservices Architecture
- Break monolith into small, independently deployable services
- Each service: owns its data store, deployed independently
- Communication: REST (API Gateway), events (SQS/SNS/EventBridge), gRPC
- Orchestration: ECS/EKS for container management
- Benefits: independent scaling, independent deployments, fault isolation

---

## Decoupling Patterns

### Why Decouple?
- Eliminate tight dependencies between components
- Allow independent scaling
- Improve resilience (failure in one component doesn't cascade)

### Decoupling Services

| Pattern | Implementation |
|---------|---------------|
| Queue-based decoupling | SQS between producer and consumer |
| Event-driven decoupling | SNS/EventBridge for pub/sub |
| Async API calls | SQS DLQ for failed messages |
| Buffer for spiky traffic | SQS queue absorbs load spikes |
| Fan-out | SNS → multiple SQS queues |

---

## Caching Strategies

### Where to Cache

| Level | Service | What's Cached |
|-------|---------|--------------|
| **Client/Browser** | CloudFront (TTL) | Static content at edge |
| **Application** | ElastiCache (Redis/Memcached) | DB query results, sessions |
| **API** | API Gateway cache | API responses |
| **Database** | DAX | DynamoDB reads |

### When to Cache
- Data doesn't change frequently
- Query is expensive (complex joins, large datasets)
- Same data requested by many users
- Low tolerance for staleness is acceptable

### Cache Invalidation Strategies
- **TTL-based expiry** — cached data expires after set time
- **Write-through** — update cache when DB updates (fresh, but write penalty)
- **Cache-aside (Lazy Loading)** — cache miss → fetch from DB → populate cache

---

## Cost Optimization Strategies

### Compute
- Use **Savings Plans** for steady-state workloads (up to 72% vs On-Demand)
- Use **Spot Instances** for fault-tolerant batch jobs (up to 90% off)
- Use **Reserved Instances** for 1-3 year committed workloads (up to 75% off)
- Right-size with **AWS Compute Optimizer** recommendations
- Use **Graviton (arm64)** instances for up to 40% better price/performance
- Use **Lambda/Fargate** for variable/spiky workloads (pay per use)

### Storage
- Use **S3 Intelligent-Tiering** for unknown access patterns
- Use **S3 Lifecycle Policies** to transition to IA/Glacier
- Use **EBS gp3** (can set IOPS independently vs gp2 where IOPS scale with size)
- Use **S3 Glacier Deep Archive** for long-term archive (lowest cost)

### Database
- Use **Aurora Serverless v2** for variable DB workloads
- Use **DynamoDB On-Demand** for unpredictable traffic; **Provisioned** with auto-scaling for predictable
- Use **ElastiCache** to reduce DB query load

### Network
- Use **VPC Gateway Endpoints** (free) for S3/DynamoDB access from VPC
- Minimize data transfer costs by keeping traffic within same region/AZ
- Use **CloudFront** to reduce origin hits and data transfer costs

### Monitoring
- **Cost Explorer** — visualize and understand spending
- **AWS Budgets** — set spending alerts
- **Trusted Advisor** — identify waste (unattached EBS, idle EC2, underutilized RIs)
- **AWS Cost Allocation Tags** — track costs by project/team/environment

---

## EC2 Pricing Decision Matrix

| Scenario | Best Option |
|----------|-------------|
| Testing/development, short-term, unpredictable | On-Demand |
| 24/7 steady-state production workload, 1-3 years | Reserved Instances or Savings Plans |
| Batch processing, can tolerate interruption | Spot Instances |
| Database on dedicated hardware (compliance) | Dedicated Host |
| Container or Lambda workloads, variable traffic | Fargate / Lambda (Savings Plans apply) |
| Consistent compute across EC2/Fargate/Lambda | Compute Savings Plans |

---

## Storage Decision Matrix

| Use Case | Service |
|----------|---------|
| Object storage, backups, web content | S3 Standard |
| Infrequent access (>30 days) | S3 Standard-IA |
| Single AZ infrequent access (lower cost) | S3 One Zone-IA |
| Unknown/changing access patterns | S3 Intelligent-Tiering |
| Long-term archive, occasional access | S3 Glacier Instant Retrieval |
| Long-term archive, flexible retrieval | S3 Glacier Flexible Retrieval |
| Coldest archive (retrieval in hours) | S3 Glacier Deep Archive |
| High-performance, single AZ | S3 Express One Zone |
| Block storage for EC2 (general) | EBS gp3 |
| Block storage for EC2 (IOPS-intensive) | EBS io2 |
| Shared file system (Linux, multi-instance) | EFS |
| Shared file system (Windows/SMB) | FSx for Windows |
| HPC/ML high-performance storage | FSx for Lustre |

---

## Database Decision Matrix

| Use Case | Service |
|----------|---------|
| Relational DB, managed OLTP | RDS |
| High-performance relational | Aurora |
| Global relational DB | Aurora Global |
| Serverless relational | Aurora Serverless v2 |
| NoSQL key-value, millisecond scale | DynamoDB |
| NoSQL global, multi-master | DynamoDB Global Tables |
| In-memory caching, rich data structures | ElastiCache Redis |
| In-memory caching, simple horizontal scale | ElastiCache Memcached |
| Data warehouse (analytics, OLAP) | Redshift |
| Query S3 without ETL | Redshift Spectrum / Athena |
| Graph database | Neptune |
| MongoDB-compatible document DB | DocumentDB |
| Immutable audit ledger | QLDB |

---

## Networking Decision Matrix

| Problem | Solution |
|---------|---------|
| Private instances need internet | NAT Gateway |
| Access S3/DynamoDB from VPC (private) | VPC Gateway Endpoint (free) |
| Access other AWS services from VPC (private) | VPC Interface Endpoint |
| Connect 2 VPCs | VPC Peering |
| Connect many VPCs + on-premises | Transit Gateway |
| On-premises to AWS (private, high bandwidth) | Direct Connect |
| On-premises to AWS (quick setup, over internet) | Site-to-Site VPN |
| Global CDN for static content | CloudFront |
| Improve routing for dynamic global content | Global Accelerator |
| Intelligent HTTP routing, Layer 7 | ALB |
| Extreme TCP/UDP performance, static IP | NLB |
| Third-party virtual appliances (firewall, IDS) | GWLB |
| DNS routing and health checks | Route 53 |

---

## Common Exam Scenario Patterns

### HA Web Application
- Multi-AZ + Auto Scaling + ALB
- RDS Multi-AZ
- S3 + CloudFront for static assets
- ElastiCache for session/query caching
- Route 53 health checks

### Global Application
- CloudFront (static), Global Accelerator (dynamic)
- Aurora Global Database or DynamoDB Global Tables
- Route 53 latency/geolocation routing
- Multiple regions with same stack

### Cost-Optimized Batch Processing
- Spot Instances (up to 90% discount)
- SQS queue to buffer work
- Auto Scaling based on queue depth
- S3 for input/output
- Terminate instances after processing

### Real-Time Data Processing
- Kinesis Data Streams → Lambda/Kinesis Data Analytics
- Kinesis Firehose → S3/Redshift
- EventBridge for event routing
- DynamoDB for results storage
- QuickSight for visualization

### Serverless API
- API Gateway → Lambda → DynamoDB
- CloudFront for caching
- Cognito for auth
- S3 + CloudFront for frontend
- X-Ray for tracing

### Hybrid Cloud
- Direct Connect (primary) + Site-to-Site VPN (backup)
- Storage Gateway (File, Volume, or Tape)
- AWS DataSync for file migration
- Active Directory integration via AWS Directory Service

---

## Well-Architected Tool

- Free tool in AWS console to review workloads against 6 pillars
- Generates improvement plan with recommendations
- Identifies high-risk issues (HRIs)
- Can share reports with AWS Partners

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — Security pillar: IAM, KMS, WAF, GuardDuty
- [[AWS-SAA-02-Compute]] — Reliability: Auto Scaling, EC2 pricing decisions
- [[AWS-SAA-03-Storage]] — Storage decision matrix, S3 lifecycle, cost optimization
- [[AWS-SAA-04-Databases]] — Database decision matrix, DR with Aurora Global
- [[AWS-SAA-05-Networking-and-CDN]] — VPC architecture, CloudFront, networking decisions
- [[AWS-SAA-06-Messaging-and-Integration]] — Decoupling patterns, event-driven architecture
- [[AWS-SAA-07-Monitoring-and-Management]] — Operational Excellence: CloudWatch, CloudTrail, Config

*AWS SAA-C03/C04 Reference*
