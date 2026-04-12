---
title: "AWS SAA — Storage"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Storage

## Amazon S3 (Simple Storage Service)

### Overview
- Object storage — stores files (objects) up to **5TB** per object
- Files stored in **buckets** with globally unique names
- Virtually unlimited number of files
- Successful uploads return **HTTP 200**
- **Strong read-after-write consistency** for all operations (since December 2020)
- All newly created buckets are **private by default**

### S3 Key Details
- Objects have: key (name), value (data), version ID, metadata, sub-resources (ACLs, torrents)
- Charges: storage size, number of requests, storage management (tier), data transfer out, Transfer Acceleration, CRR
- Bucket policies = bucket-level security; ACLs = object-level security
- Static website hosting requires `index.html` and `error.html`
- S3 is not suitable for OS or database hosting (use EBS/EFS/RDS instead)

### S3 Storage Classes

| Class | Availability | Use Case | Retrieval |
|-------|-------------|---------|-----------|
| **S3 Standard** | 99.99% | Frequently accessed data | Immediate |
| **S3 Standard-IA** | 99.9% | Infrequent access, rapid retrieval | Immediate (retrieval fee) |
| **S3 One Zone-IA** | 99.5% | Infrequent, lower cost, single AZ | Immediate (retrieval fee) |
| **S3 Intelligent-Tiering** | 99.9% | Unknown/changing access patterns | Immediate |
| **S3 Glacier Instant Retrieval** | 99.9% | Archive, millisecond access | Milliseconds |
| **S3 Glacier Flexible Retrieval** | 99.99% | Archive, flexible retrieval | 1-5 min (Expedited), 3-5 hr (Standard), 5-12 hr (Bulk) |
| **S3 Glacier Deep Archive** | 99.99% | Long-term archive, lowest cost | 12 hours (Standard), 48 hours (Bulk) |
| **S3 Express One Zone** | High | Ultra-high performance (single AZ) | Single-digit milliseconds |

- **All classes except Reduced Redundancy Storage** have **11 9s (99.999999999%)** durability
- Intelligent-Tiering: uses ML/AI to move data between tiers automatically; no retrieval fees, monitoring fee per object

> **Exam Tip:** Glacier Expedited retrievals can take 1-5 minutes but may be slower during high demand. Purchase **Provisioned Capacity** to guarantee Expedited retrieval times under all circumstances.

### S3 Versioning
- Stores all versions including writes and deletes
- Once enabled, **cannot be disabled** — only suspended
- MFA Delete for additional layer of protection
- Deleted objects get a **delete marker** — not truly deleted; delete markers are NOT replicated in CRR

> **Exam Tip:** Versioning cannot be disabled once enabled, only suspended. Existing objects get null version ID when versioning is first enabled.

### S3 Lifecycle Management
- Automates moving objects between storage tiers
- Works with versioning — apply to current and previous versions
- Can expire objects (delete after N days) or transition to cheaper storage class

### S3 Cross-Region Replication (CRR)
- **Requires versioning enabled** on both source and destination buckets
- Pre-existing objects are NOT replicated — only new uploads after CRR is enabled
- Can change ownership and storage tier for replicated objects
- **Delete markers are NOT replicated** (protects against accidental/malicious deletion)
- Can replicate to different AWS account

> **Exam Tip:** CRR does NOT replicate existing data — only new uploads after enabling. Delete markers are not replicated to the destination bucket.

### S3 Transfer Acceleration
- Uses **CloudFront edge locations** to accelerate uploads
- Upload to a distinct URL (edge location) instead of the bucket directly
- Data then travels over AWS backbone (faster than public internet)

### S3 Multipart Upload
- Recommended for files **over 100MB**; **required for files over 5GB**
- Uploads parts in parallel for improved throughput
- Can pause and resume; retry failed parts without re-uploading all
- **Byte-range fetches** for parallel downloads (failure isolated to specific byte range)

### S3 Pre-signed URLs
- Grant time-limited access to private S3 objects
- Requires: security credentials, bucket, object key, HTTP method, expiration date/time
- Anyone with the URL can access the object during validity period
- Good for: temporary downloads, upload delegation

### S3 Select
- Pull only specific data from an object using simple SQL expressions
- Improves performance up to **400%** and reduces cost
- Avoids downloading/processing entire object
- Also works with **S3 Glacier**

### S3 Encryption

| Type | Description | Key Management |
|------|-------------|---------------|
| **SSE-S3** | S3 manages keys automatically; AES-256 | AWS-managed |
| **SSE-KMS** | AWS KMS manages keys; audit trail in CloudTrail | Customer + AWS |
| **SSE-C** | Customer provides own keys; S3 handles encryption | Customer |
| **DSSE-KMS** | Dual-layer server-side encryption with KMS | Customer + AWS |
| **Client-side** | Encrypt before uploading | Customer |

- **In transit**: always SSL/TLS
- Default encryption can be set at bucket level (SSE-S3 or SSE-KMS)

### S3 Event Notifications
- Triggers on: new object added, object deleted, object restored, replication events
- Destinations: **SNS**, **SQS**, **Lambda**
- Use EventBridge for more advanced routing (filter, multiple destinations)

### S3 Static Website Hosting
- Requires: `index.html` (root), `error.html`
- Creates a publicly accessible website endpoint
- Enable public access; add bucket policy for public reads

### S3 Access Points
- Simplify access management for shared datasets
- Create named endpoints with their own policies
- Can be VPC-restricted or internet-accessible

### S3 Object Lambda
- Transform S3 data as it's retrieved
- Run Lambda function to modify object before returning to caller
- Use cases: redact PII, watermark images, convert formats

### S3 Performance
- **3,500 PUT/COPY/POST/DELETE** requests/sec per prefix
- **5,500 GET/HEAD** requests/sec per prefix
- **Optimize with prefixes** — use random/hashed prefixes for high request rates
- Use CloudFront to reduce direct GET requests to S3

### S3 Server Access Logging
- Detailed records for requests to a bucket
- Logs shipped to separate S3 bucket (same region)
- Useful for security, auditing, understanding usage

### S3 Storage Lens
- Organization-wide visibility into S3 usage and activity

### S3 Cross-Account Sharing (3 methods)
1. IAM & Bucket Policies — programmatic access, entire bucket
2. ACLs & Bucket Policies — programmatic access, individual objects
3. Cross-account IAM roles — console AND terminal access

---

## Amazon EBS (Elastic Block Store)

### Overview
- **Block-level storage** attached to a single EC2 instance
- Persists independently from EC2 instance lifecycle
- Automatically replicated within its AZ
- **EBS volume = same AZ as EC2 instance** — cannot span AZs
- 99.999% SLA

### EBS Volume Types

| Type | IOPS | Throughput | Use Case |
|------|------|-----------|---------|
| **gp3** (General Purpose SSD) | 3,000–16,000 | 125–1,000 MB/s | Most workloads (recommended) |
| **gp2** (General Purpose SSD, legacy) | Up to 16,000 | Up to 250 MB/s | Older workloads |
| **io2 Block Express** (Provisioned IOPS) | Up to 256,000 | Up to 4,000 MB/s | Mission-critical, low latency |
| **io1** (Provisioned IOPS, legacy) | Up to 64,000 | Up to 1,000 MB/s | High IOPS legacy |
| **st1** (Throughput Optimized HDD) | Up to 500 | Up to 500 MB/s | Big data, data warehouses, log processing |
| **sc1** (Cold HDD) | Up to 250 | Up to 250 MB/s | Infrequently accessed data, lowest cost |

- **gp3** is the recommended default (decoupled IOPS from storage size)
- **io2** supports **Multi-Attach** — attach to multiple EC2 instances simultaneously (same AZ, Nitro-based)
- **SSD rule**: IOPS-heavy workloads → SSD; **HDD rule**: throughput-heavy → HDD

> **Exam Tip:** gp3 allows you to provision IOPS independently from storage size (unlike gp2 where IOPS scale with size). This makes gp3 more cost-effective.

### EBS Snapshots
- Point-in-time backups stored in S3 (managed by AWS)
- **Incremental** — only changed blocks since last snapshot
- Constrained to the region where created
- Can be copied to other regions
- First snapshot captures entire volume; subsequent snapshots capture only changes
- Snapshots are asynchronous — volume can be used during snapshot
- Best practice: stop instance before snapshotting root device
- **Cannot delete** an EBS snapshot that is used by a registered AMI

### EBS Encryption
- Uses KMS customer master keys (AES-256)
- Encrypts: data at rest, data in transit between volume and instance, all snapshots, all volumes created from snapshots
- Snapshots of encrypted volumes are encrypted; can only share **unencrypted** snapshots
- Cannot change encryption state of existing volume — create encrypted copy

### EBS Root Device Storage
- **EBS-backed**: root volume survives instance termination (by default, deleted; configurable)
- **Instance Store-backed**: root volume is **ephemeral** — deleted on instance termination
- Instance store: very high IOPS, but no persistence — good for temp storage, caches, buffers
- Additional instance store volumes must be added at launch; cannot add later

### EBS Snapshots Archive
- Cost-effective long-term storage for EBS snapshots (75% cheaper)
- Retrieval time: 24-72 hours

---

## Amazon EFS (Elastic File System)

### Overview
- **Fully managed elastic NFS file system** for Linux
- Can be attached to **multiple EC2 instances simultaneously** (unlike EBS)
- Scales automatically — pay only for storage used; no pre-provisioning
- Supports thousands of concurrent NFS connections
- **Read-after-write consistency**
- Data stored across **multiple AZs** in a region

### EFS Details
- Uses **NFSv4 protocol** — open port 2049 in security group
- Best for file storage accessed by a fleet of servers
- Can access from on-premises via Direct Connect or VPN
- Can be used with: EC2, ECS, EKS, Lambda, Fargate
- **EFS One Zone** — single AZ, lower cost (20% less than Standard)
- **EFS Replication** — cross-region data protection

### EFS Performance Modes
- **General Purpose** — low latency; web servers, CMS, home directories
- **Max I/O** — higher throughput, higher latency; big data, media processing

### EFS Throughput Modes
- **Bursting** — throughput scales with storage size
- **Provisioned** — independently configure throughput regardless of size
- **Elastic** — automatically scales throughput up/down based on workload

### EFS Storage Classes
- **Standard** — frequently accessed
- **Standard-IA** — infrequently accessed (lower cost + retrieval fee)
- **One Zone** — single AZ, frequently accessed
- **One Zone-IA** — single AZ, infrequently accessed

---

## Amazon FSx for Windows

- Fully managed **native Microsoft Windows** file system (SMB protocol)
- Built on Windows Server; supports Microsoft AD authentication
- Use when: Windows apps need SMB-based file storage
- Encrypted at rest and in transit automatically
- Deploy in single AZ or **Multi-AZ** configuration
- Storage: SSD or HDD
- Daily automated backups; admin snapshots
- Deduplication and compression built-in
- Access from: EC2, WorkSpaces, on-premises (via VPN/Direct Connect)

> **Exam Tip:** Need Windows/SMB file shares → FSx for Windows. Need Linux/NFS shared storage → EFS. Need high-performance Linux computing → FSx for Lustre.

---

## Amazon FSx for Lustre

- High-performance file system for **HPC, ML, video processing, financial modeling**
- Processes datasets at up to hundreds of GB/s throughput, millions of IOPS, sub-millisecond latency
- Compatible with Linux-based AMIs (Amazon Linux, RHEL, CentOS, Ubuntu, SUSE)
- Can **read/write directly to/from S3** — seamless integration
- Good for: compute clusters, ML training, simulation workloads

---

## AWS Storage Gateway

### Overview
- Bridges on-premises environments with AWS cloud storage
- Physical device or VM image deployed in on-premises data center
- Supports VMware ESXi and Microsoft Hyper-V
- All S3 features available: versioning, lifecycle, CRR, bucket policies

### Gateway Types

| Type | Protocol | Backend | Use Case |
|------|----------|---------|---------|
| **File Gateway** | NFS/SMB | S3 | NFS/SMB mount point for S3 |
| **Volume Gateway** | iSCSI | S3/EBS | Virtual hard drives in cloud |
| **Tape Gateway** | iSCSI (VTL) | S3/Glacier | Replace physical tape library |

### Volume Gateway: Stored vs Cached

| | Stored Volumes | Cached Volumes |
|-|---------------|----------------|
| Primary data | On-premises | AWS (S3) |
| Cached data | Full dataset local | Most frequently used only |
| AWS role | Backup/secondary | Primary |
| Latency | Low (all data local) | Low for frequently accessed |

> **Exam Tip:** Stored Volumes = all data on-prem, S3 is backup. Cached Volumes = AWS is primary, hot data cached locally. Stored = lower latency for all data; Cached = reduces need to scale on-prem infrastructure.

---

## AWS Snow Family

### Snowball
- Physical data transfer device for migrating large amounts of data to AWS
- Use when: >1 week to upload via internet, physically isolated environment, expensive internet
- Rule of thumb: if transfer takes >1 week via spare internet capacity, consider Snowball
- Secure: tamper-evident, encrypted; wiped after data transfer complete

### Snowball Edge
- Snowball **with compute capabilities** (Lambda + EC2 instance types)
- Run code while data is in transit or at remote locations
- Use cases: airplanes, remote offices, disconnected environments
- Can be **clustered locally** for better performance
- Does not need to be limited to data transfer — can serve as edge compute

### Snowmobile
- **Exabyte-scale** data transfer — 100 petabytes per Snowmobile
- 45-foot shipping container hauled by semi-truck
- Use for: moving entire data centers to cloud

### Snow Family Decision
| Data Volume | Solution |
|-------------|---------|
| Terabytes to petabytes | Snowball |
| Petabytes+ with compute | Snowball Edge |
| Exabytes (100+ PB) | Snowmobile |

---

## Storage Decision Matrix

| Use Case | Recommended Service |
|----------|---------------------|
| Object storage, backups, static content | S3 |
| OS/boot volume for EC2 | EBS (gp3) |
| Shared file system for Linux instances | EFS |
| Shared file system for Windows instances | FSx for Windows |
| HPC / ML high-performance storage | FSx for Lustre |
| Hybrid on-prem + cloud storage | Storage Gateway |
| Move large data volumes offline | Snow Family |
| Ephemeral high-IOPS scratch storage | EC2 Instance Store |
| Data warehouse storage | Redshift |

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — S3 bucket policies, ACLs, KMS encryption, Macie
- [[AWS-SAA-02-Compute]] — EBS volumes for EC2, instance store vs EBS
- [[AWS-SAA-04-Databases]] — RDS on EBS, Aurora storage architecture
- [[AWS-SAA-05-Networking-and-CDN]] — CloudFront with S3 origin, Transfer Acceleration
- [[AWS-SAA-06-Messaging-and-Integration]] — S3 event notifications to SQS/SNS/Lambda
- [[AWS-SAA-07-Monitoring-and-Management]] — S3 server access logging, CloudTrail for S3 data events
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — Storage decision matrix, cost optimization

*AWS SAA-C03/C04 Reference*
