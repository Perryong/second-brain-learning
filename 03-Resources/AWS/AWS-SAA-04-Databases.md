---
title: "AWS SAA — Databases"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Databases

## Amazon RDS (Relational Database Service)

### Overview
- Managed relational database service — AWS handles patching, backups, hardware provisioning
- **You cannot SSH into RDS instances** — AWS manages the underlying OS
- Not serverless (except Aurora Serverless) — runs on virtual machines

### Supported Engines
- MySQL, PostgreSQL, MariaDB, Oracle, Microsoft SQL Server, **Aurora**, SAP HANA

### OLTP vs OLAP
- **OLTP (Online Transaction Processing)** — core app logic, frequent reads/writes → RDS, Aurora
- **OLAP (Online Analytical Processing)** — complex queries for insights → Redshift (not RDS)

> **Exam Tip:** Use RDS for OLTP workloads. Use Redshift for OLAP/analytics. Never use RDS for analytics at petabyte scale.

### RDS Multi-AZ
- Automatically creates a **synchronous standby** replica in a different AZ
- Failover: RDS updates DNS endpoint automatically — no manual intervention
- **Recovery, not performance** — standby cannot serve read traffic
- Backups taken from the standby instance
- Supported for all engines **except Aurora** (Aurora is natively fault-tolerant)
- Can force failover by rebooting primary instance
- Multi-AZ spans AZs within a region, **not across regions**

> **Exam Tip:** Multi-AZ = high availability/disaster recovery. Read Replicas = performance/read scaling. They are different features for different purposes.

### RDS Read Replicas
- **Asynchronous replication** — may have slight lag
- For **performance enhancement** — offload read queries from primary
- Up to **5 read replicas** per DB instance
- Each replica has its own DNS endpoint
- Can be in same AZ, different AZ, or **different region** (cross-region)
- Can have read replicas of read replicas (watch for latency/lag)
- **No automatic failover** — must manually promote to master if primary fails
- **Automated backups must be enabled** to use read replicas
- Can be promoted to standalone DB instance

> **Exam Tip:** Read Replicas require automated backups to be enabled. They do NOT provide automatic failover — that's Multi-AZ's job.

### RDS Backups
- **Automated Backups** — daily full backup + transaction logs throughout day; point-in-time recovery (1-35 days retention); **enabled by default**; stored in S3
  - Free storage up to size of your DB
  - Deleted when RDS instance is deleted
- **DB Snapshots** — manual; retained even after RDS instance is deleted; must be explicitly deleted

> **Exam Tip:** Automated backups are deleted when you delete the RDS instance. Manual snapshots persist. When deleting RDS, AWS asks if you want a final snapshot.

### RDS Security
- **IAM Database Authentication** — for MySQL and PostgreSQL; use auth token instead of password; tokens valid 15 minutes; traffic encrypted via SSL
- **Encryption at rest** — KMS-based AES-256; must enable at creation time (cannot enable after)
- Encrypted instances: all backups, read replicas, snapshots also encrypted
- Cannot modify an encrypted instance to unencrypted
- **RDS Proxy** — connection pooling, reduce DB connection overhead; integrates with Secrets Manager

### RDS Enhanced Monitoring
- Metrics from **agent on the DB instance** (vs. CloudWatch which gets metrics from hypervisor)
- Real-time OS metrics; stored in CloudWatch Logs for 30 days
- More granular than standard CloudWatch (hypervisor-level vs. OS-level)

### Secrets Manager Integration
- Store and automatically rotate DB credentials
- Lambda function handles rotation; applications retrieve credentials via API call
- Eliminates hardcoded credentials in code

---

## Amazon Aurora

### Overview
- AWS's flagship DB — **MySQL and PostgreSQL compatible**
- Up to **5x faster** than MySQL, **3x faster** than PostgreSQL
- Costs 1/10th of commercial enterprise databases
- 99.99% availability; **automatically fault-tolerant**

### Aurora Storage Architecture
- **Distributed storage** — 6 copies of data across 3 AZs (2 copies per AZ)
- Can handle loss of **2 copies** without impacting write availability
- Can handle loss of **3 copies** without impacting read availability
- Storage is **self-healing** — continuously scanned and repaired automatically
- Starts at 10GB, scales in 10GB increments up to **128TB** automatically

### Aurora Replication
- Up to **15 Aurora Read Replicas** (vs. 5 for RDS)
- Aurora replicas can serve as both standby (Multi-AZ) AND read endpoints
- **Automated failover** is only possible with Aurora read replicas (not with MySQL/PostgreSQL replicas)
- Can also replicate to MySQL (5 replicas) or PostgreSQL (1 replica) downstream

### Aurora Cluster Endpoints
- **Cluster endpoint (writer)** — connects to primary instance for DDL/writes
- **Reader endpoint** — load-balances reads across all Aurora Replicas
- **Custom endpoints** — create groups of instances for specific workloads
- **Instance endpoints** — connect to a specific instance (for diagnosis/tuning)
- After failover: entry point stays the same, application reconnects automatically

### Aurora Serverless v2
- On-demand, autoscaling configuration for MySQL/PostgreSQL-compatible Aurora
- Automatically starts/stops and scales based on application usage
- Ideal for: infrequent, intermittent, or unpredictable workloads
- Pay per invocation — scales in ACU (Aurora Capacity Units) increments
- Removes complexity of managing instance size

### Aurora Global Database
- Spans **multiple AWS regions** for global applications
- Primary region for writes; secondary regions for reads (up to 5 secondary regions)
- Replication lag < 1 second
- **RPO of 1 second, RTO < 1 minute** for disaster recovery
- Enables cross-region failover

> **Exam Tip:** RDS Multi-AZ = same-region HA. Aurora Global Database = cross-region HA/DR. If the question mentions global users or cross-region failover with < 1 second replication, think Aurora Global.

### Aurora vs RDS Quick Compare

| Feature | RDS | Aurora |
|---------|-----|--------|
| Storage Auto-scaling | No | Yes (to 128TB) |
| Max read replicas | 5 | 15 |
| Failover (Multi-AZ) | DNS update, ~60-120 sec | Automatic, ~30 sec |
| Multi-AZ support | Yes | Built-in |
| Cross-region | Read replicas only | Global Database |
| Serverless | No (except Aurora) | Serverless v2 |

---

## Amazon DynamoDB

### Overview
- **Key-value and document** NoSQL database
- Fully managed, multi-region, multi-master
- **Single-digit millisecond** performance at any scale
- Built-in security, backup/restore, in-memory caching
- Stored on SSD; spread across **3 geographically distinct data centers**

### DynamoDB Data Model
- **Table** → **Items** (rows) → **Attributes** (columns)
- Items can have different attributes — flexible schema
- **Partition Key (Hash Key)** — must be unique; determines partition
- **Sort Key (Range Key)** — optional; with partition key forms composite primary key
- **High cardinality** partition keys improve performance (spread I/O across partitions)

### Indexes

| Index Type | Key | Per Table | Projected Attributes | Query Flexibility |
|-----------|-----|-----------|---------------------|------------------|
| **LSI** (Local Secondary Index) | Same partition key, different sort key | Max 5 | Any | Must define at table creation |
| **GSI** (Global Secondary Index) | Different partition + sort key | Max 20 | Any | Can create anytime |

> **Exam Tip:** LSIs share the same partition key as the table. GSIs can have completely different partition and sort keys. GSIs can be created/deleted anytime; LSIs only at table creation.

### Consistency Models
- **Eventually Consistent Reads** (default) — copies are identical within ~1 second; higher throughput
- **Strongly Consistent Reads** — always returns latest data; uses more read capacity units

### Capacity Modes

| Mode | Description | Best For |
|------|-------------|---------|
| **Provisioned** | Specify read/write capacity units (RCU/WCU) | Predictable traffic; cost savings with auto-scaling |
| **On-Demand** | Pay per request; auto-scales | Unpredictable or new workloads |

### DynamoDB Accelerator (DAX)
- Fully managed, highly available **in-memory cache** for DynamoDB
- Reduces response times from **milliseconds → microseconds**
- Write-through cache (also improves write performance)
- Scale to **10-node cluster**, millions of requests/second
- Fully compatible with existing DynamoDB API calls (no code changes needed)
- HA — auto-failover to replica in another AZ
- Good for read-heavy workloads, not strongly consistent reads

> **Exam Tip:** DAX is for DynamoDB only. For RDS caching, use ElastiCache. DAX sits in front of DynamoDB tables.

### DynamoDB Streams
- Ordered log of item-level changes in a DynamoDB table
- Captures: new item, updated item (before/after images), deleted item
- Integrated with **Lambda triggers** — auto-respond to table changes
- Events appear in stream within milliseconds
- Retention: **24 hours**

### DynamoDB Global Tables
- **Multi-region, multi-master** replication
- Based on DynamoDB Streams
- Replication latency: **typically < 1 second**
- Application failover: redirect DynamoDB calls to another region
- Eliminates conflict resolution complexity

### DynamoDB TTL (Time to Live)
- Automatically delete items after a specified time
- Set an attribute with Unix epoch timestamp — items expire at that time
- Useful for session data, log entries, temporary data
- No additional cost; deleted items appear in Streams

### DynamoDB Transactions
- All-or-nothing operations across multiple items/tables
- **TransactWriteItems** / **TransactGetItems**
- Good for: financial transactions, inventory management, multi-table operations

---

## Amazon Redshift

### Overview
- Fully managed, **petabyte-scale data warehouse** service
- For **OLAP** (Online Analytical Processing) — business intelligence, analytics
- Up to **100x faster** than traditional data warehouses

### Redshift Architecture
- **Leader node** — manages client connections, receives queries, builds query plans
- **Compute nodes** — store data, execute queries (up to 128 compute nodes)
- Data distributed and queried in **Massive Parallel Processing (MPP)**
- **Columnar storage** — stores data by column (not row) → better compression, faster analytical queries
- Compressed; no indexes or materialized views needed → smaller storage footprint

### Key Details
- **Not Multi-AZ** — single AZ; for multi-AZ, spin up separate cluster or restore snapshot to new AZ
- Encrypted: **SSL in transit**, **AES-256 at rest** (AWS-managed or KMS/CloudHSM)
- Snapshots: 1-day retention default, max 35 days; manual snapshots never auto-deleted
- Can replicate snapshots to a different region asynchronously
- Billed for: compute node hours, backups, data transfer within VPC
- Default: locked down — must associate with security group to allow access

### Redshift Serverless
- Run and scale Redshift without managing clusters
- Pay only for compute used during queries

### Redshift Spectrum
- Query **exabytes of unstructured data in S3** without loading/ETL
- Cluster and S3 data must be in same region
- External S3 tables are **read-only** (no INSERT/UPDATE/DELETE)
- Processing occurs in Spectrum layer; less cluster capacity consumed

### Redshift Concurrency Scaling
- Automatically adds capacity for concurrent queries
- Near-limitless concurrent read queries

### Redshift Enhanced VPC Routing
- Forces all COPY/UNLOAD traffic through your VPC (not public internet)
- Enables use of VPC security groups, NACLs, VPC endpoints

---

## Amazon ElastiCache

### Overview
- Fully managed **in-memory caching service**
- Improves performance by serving frequently-accessed data from memory
- Supports **Redis** and **Memcached**

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Rich (strings, hashes, lists, sets, sorted sets) | Simple key-value only |
| Persistence | Yes (snapshots + AOF) | No |
| Replication | Yes (Multi-AZ, read replicas) | No |
| Clustering/sharding | Yes | Yes |
| Multi-threaded | No (single-threaded) | Yes |
| Pub/Sub messaging | Yes | No |
| Lua scripting | Yes | No |
| Best for | Complex caching, sessions, leaderboards, pub/sub | Simple caching, horizontal scaling |

> **Exam Tip:** Need HA, persistence, complex data structures, or pub/sub → Redis. Need simple, horizontal scaling, multi-threaded → Memcached.

### Caching Strategies

| Strategy | Description | Use Case |
|----------|-------------|---------|
| **Lazy Loading** | Cache only on cache miss; may return stale data | Read-heavy, tolerate stale data |
| **Write-Through** | Update cache when DB is updated; no stale data | Write-heavy, need fresh data |
| **TTL (Time-to-Live)** | Expire cache entries after set time | Balance freshness vs. performance |

### ElastiCache Use Cases
- Database query result caching (reduce DB load)
- Session management (store user sessions)
- Leaderboards / real-time analytics (sorted sets in Redis)
- Rate limiting / counters
- Pub/Sub messaging

---

## Amazon Neptune

- Fully managed **graph database** service
- Supports: Apache TinkerPop Gremlin (property graph), W3C SPARQL (RDF graph)
- Use cases: social networks, knowledge graphs, recommendation engines, fraud detection, identity graphs
- Highly available; Multi-AZ; up to 15 read replicas; automatic backups

---

## Amazon DocumentDB

- Fully managed **MongoDB-compatible** document database
- Designed for JSON data at scale
- Compatible with existing MongoDB drivers and tools
- Decoupled storage from compute; scales to millions of requests
- Automatically replicates data across 3 AZs

---

## Amazon QLDB (Quantum Ledger Database)

- Fully managed **immutable, cryptographically verifiable** ledger database
- **Append-only** — data cannot be modified or deleted
- Cryptographic hash for verifiable history
- Use cases: financial transactions, supply chain, HR records, banking ledgers
- SQL-like API (PartiQL); serverless; scales automatically

---

## Database Decision Matrix

| Use Case | Service |
|----------|---------|
| Relational DB (OLTP), managed | RDS |
| High-performance relational DB, managed | Aurora |
| Serverless relational DB | Aurora Serverless |
| Global relational DB (< 1 sec replication) | Aurora Global |
| Key-value / document (NoSQL, millisecond) | DynamoDB |
| In-memory NoSQL cache | ElastiCache (Redis or Memcached) |
| Data warehouse (OLAP, petabyte) | Redshift |
| S3 data querying (no ETL) | Redshift Spectrum |
| Graph database | Neptune |
| MongoDB-compatible | DocumentDB |
| Immutable audit ledger | QLDB |
| Self-managed SQL (full OS control) | EC2 with DB software |

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — RDS IAM auth, Secrets Manager, KMS encryption
- [[AWS-SAA-02-Compute]] — EC2 as alternative to RDS, SQS for write buffering
- [[AWS-SAA-03-Storage]] — RDS automated backups to S3, Redshift and S3 Spectrum
- [[AWS-SAA-05-Networking-and-CDN]] — RDS in private subnet, VPC for Redshift
- [[AWS-SAA-06-Messaging-and-Integration]] — DynamoDB Streams with Lambda, SQS for DB write buffering
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — DB decision matrix, caching strategies

*AWS SAA-C03/C04 Reference*
