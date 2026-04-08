---
title: "AWS SAA — Compute"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Compute

## EC2 (Elastic Compute Cloud)

### Overview
- Resizable virtual server instances in the cloud
- You pay for: **running** instances (CPU, memory, storage, network); **stopped** instances charge EBS storage only
- Billing states: `pending` (not billed) → `running` (billed) → `stopping` (not billed unless hibernating) → `stopped` (not billed) → `terminated` (not billed)
- **Reserved Instances that are terminated are still billed until end of their term**

### Instance Types (2025 Current)
| Category | Example Types | Use Case |
|----------|--------------|----------|
| General Purpose | M7g, T4g, M7i, T3, M7a | Balanced CPU/memory/networking |
| Compute Optimized | C7g, C7i, C7a, C7gd | CPU-intensive workloads |
| Memory Optimized | R7iz, X2idn, R7a, R7g, U7i | Large in-memory datasets |
| Accelerated Computing | P4d, G5, P5en, Trn2 | ML/GPU workloads |
| Storage Optimized | Im4gn, Is4gen, I4i | High IOPS storage workloads |

- **Graviton (arm64)** instances (M7g, C7g, R7g, T4g) offer better price/performance
- **Nitro System** — enhanced security and performance on modern instance types

### EC2 Pricing Models

| Model | Discount | Best For |
|-------|----------|---------|
| **On-Demand** | None (baseline) | Short-term, unpredictable workloads |
| **Reserved Instances** | Up to 75% | Steady-state, 1 or 3 year commitment |
| **Savings Plans** | Up to 72% | Flexible compute commitment (EC2, Fargate, Lambda) |
| **Spot Instances** | Up to 90% | Fault-tolerant, flexible start/end times |
| **Dedicated Hosts** | Varies | Compliance/licensing requirements |
| **Dedicated Instances** | Varies | Single-tenant hardware, no host control |
| **Capacity Reservations** | None | Reserve capacity without commitment discount |

**Important 2025 Update:**
- **Savings Plans and Reserved Instances are restricted to single end customer usage from June 1, 2025** — cannot be shared across organizations for resale

> **Exam Tip:** Spot Instances — you are NOT charged if AWS terminates the instance due to price change. You ARE charged if you terminate it yourself for any hour it ran. Use for batch processing, data analysis, flexible jobs.

### Reserved Instance Types
- **Standard RI** — 75% discount; inflexible; cannot change region; choose specific AZ or entire region
- **Convertible RI** — 54% discount; can modify instance type (but not downgrade); more flexibility
- **Scheduled RI** — reserve for specific time windows (e.g., school hours for education software)

### AMI (Amazon Machine Image)
- Blueprint for launching EC2 instances (OS, software, config)
- Can be created from EBS snapshots; can be copied across regions
- **When copying AMIs**: launch permissions, user-defined tags, and S3 bucket permissions are NOT copied automatically
- **Golden AMI** = fully customized AMI ready for immediate launch
- EBS-backed vs Instance Store-backed root volumes

### EC2 User Data
- Scripts passed to an instance at launch (run as root)
- Used for automated configuration tasks (e.g., install packages, run scripts)
- Runs once on first boot by default

### Instance Metadata (IMDSv2)
- Access instance metadata at `http://169.254.169.254/latest/meta-data/`
- **IMDSv2** (v2 preferred) — requires session-oriented requests using PUT tokens; more secure
- IMDSv1 = simple GET request (less secure, legacy)
- Metadata queries are excluded from VPC Flow Logs

> **Exam Tip:** IMDSv2 uses a session token to prevent SSRF attacks. AWS recommends enforcing IMDSv2 on all instances via instance metadata options.

### EC2 Placement Groups

| Type | Description | Use Case |
|------|-------------|---------|
| **Cluster** | All instances in single AZ, physically close | Lowest latency, highest throughput (HPC) |
| **Spread** | Each instance on distinct hardware | High availability, small number of critical instances |
| **Partition** | Multiple instances per partition, partitions isolated | Large distributed workloads (Hadoop, Kafka, Cassandra) |

- Placement group names must be unique within your AWS account
- You can move an existing (stopped) instance into a placement group via CLI/SDK (not console)
- Only certain instance types can be launched into Cluster placement groups (compute, GPU, storage, memory optimized)

> **Exam Tip:** Cluster = performance (same AZ). Spread = availability (separate hardware, different racks). Partition = balance (isolated failure domains, multiple instances per partition).

### EC2 Security
- Termination protection = **disabled by default** — enable to prevent accidental termination
- EC2 uses public-key cryptography for login (key pairs)
- EBS root volume deleted on termination by default; additional volumes are **preserved**
- Source/Destination Check: **must disable** on NAT Instances (they proxy traffic)

### ENI (Elastic Network Interfaces)
- Virtual network card for EC2 instances
- Can be hot-attached (running), warm-attached (stopped), or cold-attached (launch time)
- Fail-over: detach ENI from failed instance, reattach to standby — maintains private IP, Elastic IP, MAC address
- **Enhanced Networking ENI** — SR-IOV for higher bandwidth, lower latency, higher PPS; no extra charge
- **EFA (Elastic Fabric Adapter)** — for HPC and ML; supports OS-bypass on Linux for direct hardware access

---

## EC2 Auto Scaling

### Components
1. **Groups** — logical grouping (web server group, DB group)
2. **Launch Templates/Configurations** — define AMI, instance type, security groups, block devices
3. **Scaling Options** — condition-based, schedule-based, predictive

### Scaling Policies
- **Dynamic scaling** — responds to demand (most common)
- **Scheduled scaling** — predictable traffic spikes
- **Predictive scaling** — ML-based, pre-emptive scaling
- **Maintain current count** — always keep N instances running, replace unhealthy

### Key Details
- **Default Cooldown Period**: 300 seconds (5 min) between scaling activities
- Default termination policy: terminate instance in AZ with most instances; prefer oldest launch config; then closest to next billing hour
- Can suspend/resume scaling processes for investigation
- One launch configuration per Auto Scaling group at a time; **cannot modify** existing launch configs, must create new
- For HA: use multiple AZs and regions

---

## AWS Lambda

### Overview
- **Serverless** compute — no servers to provision or manage
- Pay only for: number of requests + compute duration (rounded to nearest 1ms)
- First 1 million requests/month free; $0.20 per million after
- Scales **horizontally and automatically** — each request maps to one function invocation

### Supported Runtimes (2025)
- Node.js 22, Python 3.13, Java 21, .NET 9, Ruby 3.4
- Custom runtimes (e.g., Go via provided.al2023)
- **Amazon Linux 2023** for newer runtimes
- **arm64 / Graviton** — improved performance and cost efficiency
- **Container images** — deploy as Docker containers (up to 10GB); familiar tooling, larger dependencies

### Event Triggers
- API Gateway (HTTP), S3 (object events), DynamoDB Streams, Kinesis, SQS, SNS, EventBridge, CloudWatch Events, Cognito, IoT

### Lambda Limits
- Memory: 128MB – 10GB
- Execution timeout: up to **15 minutes**
- Deployment package: 50MB (zipped), 250MB (unzipped), or 10GB (container image)
- Concurrent executions: **1,000 per region** (soft limit, can be increased)
- Environment variables encrypted with KMS by default

### Lambda Concurrency
- **Provisioned Concurrency** — pre-initialize execution environments; eliminates cold starts
- **Reserved Concurrency** — guarantee N executions for a function; prevents it from using all account concurrency

### Lambda@Edge
- Run Lambda functions at **CloudFront edge locations**
- Executes at 4 points: Viewer Request, Origin Request, Origin Response, Viewer Response
- Customize CloudFront content delivery without managing servers at origin
- Reduces origin infrastructure complexity

> **Exam Tip:** Lambda@Edge runs at edge locations (closest to viewer). CloudFront Functions are lighter and cheaper but more limited. Lambda@Edge supports more languages and longer execution (up to 5 sec viewer-side, 30 sec origin-side).

### Lambda in VPC
- Must provide VPC subnet IDs and security group IDs
- Lambda creates ENIs in your VPC to connect to private resources
- Ensure NAT Gateway exists if Lambda needs internet access from inside VPC

---

## Amazon ECS (Elastic Container Service)

### Overview
- Fully managed **Docker container** orchestration service
- Manages: service discovery, load balancing, health monitoring, auto scaling

### Launch Types

| Launch Type | Description | Use Case |
|-------------|-------------|---------|
| **EC2** | You manage the EC2 instances (host fleet) | Cost control, custom config |
| **Fargate** | Serverless — AWS manages infrastructure | Simplicity, no instance management |
| **ECS Anywhere** | Run containers on-premises or other clouds | Hybrid deployments |

- **ECS Anywhere** — extends ECS to your own infrastructure
- Supports Windows Containers
- AWS Copilot CLI simplifies ECS deployments
- Deep integration with ALB/NLB, IAM, EBS, EFS, CloudWatch

> **Exam Tip:** Fargate = serverless containers (no EC2 management). EC2 launch type = you manage the hosts. Choose Fargate for simplicity; EC2 for cost optimization at large scale or custom hardware requirements.

### ECS Cluster Components
- **Task Definition** — blueprint for containers (image, CPU, memory, ports, env vars)
- **Task** — running instance of a task definition
- **Service** — maintains desired task count, integrates with load balancers
- **Cluster** — logical grouping of EC2 instances or Fargate resources

---

## Amazon ECR (Elastic Container Registry)

- Fully managed Docker/OCI-compatible container image registry
- Integrated with ECS and EKS
- Features: image tagging, versioning, replication, **vulnerability scanning** (via Inspector), lifecycle management
- Supports IPv6
- **Pull Through Cache rules** — automatically cache images from public registries (Docker Hub, ECR Public)

---

## Amazon EKS (Elastic Kubernetes Service)

### Overview
- Fully managed **Kubernetes** service
- AWS manages the control plane (API server, etcd); you manage worker nodes (or use Fargate)
- Supports Kubernetes 1.33 (as of May 2025)
- Certified Kubernetes conformant — migrate standard K8s apps without refactoring

### Key Features
- **EKS Dashboard** — centralized visibility and management
- **EKS Pod Identity** — simplifies IAM credentials for pods (replaces IRSA in some cases)
- **EKS Anywhere** — deploy EKS on-premises or in other clouds
- Load balancing: ALB, NLB, CLB via AWS Load Balancer Controller
- Monitoring: CloudWatch Container Insights

> **Exam Tip:** EKS = managed Kubernetes. ECS = managed Docker (simpler). Choose EKS if you need Kubernetes compatibility or already use K8s. Choose ECS for simpler AWS-native container management.

---

## AWS Elastic Beanstalk

- **PaaS** — deploy applications without managing infrastructure
- Just upload code; Beanstalk handles: capacity provisioning, auto scaling, LB, health monitoring, deployments
- Supports: Java, .NET, PHP, Node.js, Python, Ruby, Go, Docker
- Updates via **duplicate environment swap** — no downtime; can roll back if new version fails
- Best for developers who want simplicity over control

---

## AWS Batch

- **Fully managed batch processing** at any scale
- Automatically provisions optimal compute resources based on volume and resource requirements
- Uses EC2 and Spot Instances for cost efficiency
- Integrates with S3, DynamoDB, CloudWatch
- Good for: ML model training, video transcoding, genomics, financial simulations

---

## AWS Lightsail

- Simplified cloud platform for simple workloads
- Fixed-price bundles (compute + storage + transfer)
- Good for: simple websites, blogs, small apps, dev/test environments
- Includes: VMs, containers, databases, CDN, load balancers, DNS
- Designed for those with limited cloud experience

---

## EC2 Pricing Decision Matrix

| Scenario | Recommended Pricing |
|----------|---------------------|
| Short-term, unpredictable, can't be interrupted | On-Demand |
| Steady-state, predictable 1-3 year usage | Reserved Instances / Savings Plans |
| Flexible timing, fault-tolerant, interruptible | Spot Instances |
| Dedicated physical server for compliance/licensing | Dedicated Host |
| No minimum cost, pay per use, serverless | Lambda / Fargate |

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — IAM roles for EC2/Lambda, instance profiles, STS
- [[AWS-SAA-03-Storage]] — EBS volumes, instance store vs EBS, S3 for Lambda
- [[AWS-SAA-05-Networking-and-CDN]] — VPC, security groups, placement groups, ALB/NLB
- [[AWS-SAA-06-Messaging-and-Integration]] — Lambda triggers from SQS/SNS/Kinesis/EventBridge
- [[AWS-SAA-07-Monitoring-and-Management]] — CloudWatch for EC2 metrics, Auto Scaling, Systems Manager
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — EC2 pricing decisions, serverless patterns

*AWS SAA-C03/C04 Reference*
