---
title: "AWS SAA — Messaging and Integration"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Messaging and Integration

## Amazon SQS (Simple Queue Service)

### Overview
- Fully managed **message queue** service — decouple components and scale independently
- **Pull-based** (not push) — consumers poll the queue for messages
- Stores messages while waiting for another service to process them
- Can contain **unlimited number of messages**

### Queue Types

| Feature | Standard Queue | FIFO Queue |
|---------|----------------|-----------|
| Ordering | Best-effort (may receive out of order) | Strict FIFO (guaranteed order) |
| Delivery | At least once (may get duplicates) | **Exactly-once processing** |
| Throughput | Nearly unlimited TPS | 300 TPS (up to 3,000 with batching) |
| Use case | High-throughput, order doesn't matter | Order and deduplication critical |

> **Exam Tip:** Standard queue = at-least-once delivery (duplicates possible). FIFO = exactly-once, strictly ordered, but lower throughput. FIFO queue names must end in `.fifo`.

### SQS Key Details
- **Message retention**: 1 minute to **14 days** (default: 4 days)
- **Maximum message size**: 256KB
- **Visibility timeout**: time a message is hidden after a consumer reads it (max 12 hours)
  - If not deleted within timeout, message becomes visible again (can lead to duplicate processing)
  - To reduce duplicates: increase visibility timeout
  - Consumer must explicitly delete message after processing
- **Delay queues**: delay delivery of new messages for up to 15 minutes (default: 0)
- **Long polling**: wait up to 20 seconds for a message; reduces empty responses and costs
- **Short polling** (default): returns immediately, even if empty — more frequent API calls

### Dead Letter Queue (DLQ)
- After a message fails processing N times (maxReceiveCount), it's moved to DLQ
- Use for debugging failed messages; set appropriate retention on DLQ
- **DLQ must be same type as source queue** (Standard → Standard DLQ, FIFO → FIFO DLQ)

### SQS Long Polling vs Short Polling
- **Short polling** (default): `ReceiveMessageWaitTimeSeconds = 0` — returns immediately
- **Long polling**: `ReceiveMessageWaitTimeSeconds > 0` — waits up to 20 sec for message
- Long polling: fewer API calls, lower cost, reduced CPU usage

### SQS Batch Operations
- Send, receive, or delete messages in batches of up to 10
- Reduces API calls and costs

### SQS Fan-Out Pattern
- Combine **SNS + SQS**: publish to SNS topic → fan out to multiple SQS queues
- Each queue has its own independent subscribers/processing
- Common architecture for event-driven fan-out

---

## Amazon SNS (Simple Notification Service)

### Overview
- Fully managed **pub/sub messaging** service
- **Push-based** — messages pushed to subscribers immediately (no polling)
- High-throughput, many-to-many messaging

### SNS Key Details
- **Topics** — channels for messages; publishers send to topics; subscribers receive from topics
- One topic supports many subscriber types and many endpoints
- Messages stored redundantly across multiple AZs
- No long/short polling — instantaneous push

### Subscription Endpoints
- **SQS** — fan-out to queues
- **Lambda** — trigger functions
- **HTTP/S** — webhook callbacks
- **Email/Email-JSON** — notify humans
- **SMS** — text messages
- **Mobile push** — Apple (APNs), Google (FCM/GCM), Amazon, Windows
- **Kinesis Data Firehose** — stream to S3/Redshift/OpenSearch

### SNS FIFO Topics
- Strict ordering + deduplication for ordered notification flows
- Higher throughput than SQS FIFO queues in fan-out architectures
- Only SQS FIFO queues can subscribe to SNS FIFO topics

### Message Filtering
- Apply filter policies to subscriptions — subscribers only receive messages that match
- JSON-based attribute matching
- Reduces unnecessary message processing

### SNS Message Encryption
- Encrypted at rest (SSE) using KMS
- Encrypted in transit (HTTPS)

---

## Amazon Kinesis

### Overview
- Fully managed **real-time streaming data** service
- Ingest real-time data: IoT, clickstreams, application logs, financial transactions, video, audio

### Kinesis Data Streams

- **Real-time ingestion and processing** of streaming data
- Data retained for **24 hours by default** (up to **365 days**)
- Data contained within **shards**
- Each shard: 1MB/s input, 2MB/s output
- Total capacity = sum of all shard capacities
- **Partition keys** determine which shard receives data; maintains order within a shard
- Consumers (EC2, Lambda, Kinesis Data Analytics) read from shards
- Encrypted at rest (KMS) and in transit (HTTPS)

> **Exam Tip:** Kinesis Data Streams = real-time, custom processing, multi-consumer, data retained for replay. SQS = queuing, at-least-once delivery, 14-day retention. Choose Kinesis for real-time analytics on streaming data.

### Kinesis Data Firehose

- **Deliver streaming data** to data lakes and analytics tools
- **No persistent storage** — analyze and deliver immediately
- Can transform data with Lambda before delivery
- Destinations: **S3**, **Redshift**, **OpenSearch (Elasticsearch)**, **Splunk**, **HTTP endpoint**, **3rd-party tools**
- Near-real-time (buffers: 60-900 seconds or 1MB-128MB)
- Fully managed, serverless; no shards to manage

> **Exam Tip:** Firehose = data delivery pipeline (not analytics). Data Analytics = analysis. Streams = raw streaming with custom processing. Firehose automatically loads to S3/Redshift/OpenSearch.

### Kinesis Data Analytics
- Analyze streaming data in real time using **Apache Flink** or **SQL**
- Works with both Kinesis Data Streams and Kinesis Firehose as sources
- Serverless; results sent to another stream, Firehose, or Lambda
- Use for: real-time dashboards, anomaly detection, real-time metrics

### Kinesis Video Streams
- Securely stream video from connected devices to AWS
- For ML, analytics, playback
- Integrates with Rekognition for video analysis

### Kinesis Decision

| Scenario | Service |
|----------|---------|
| Real-time processing with custom consumers | Kinesis Data Streams |
| Deliver streaming data to S3/Redshift/OpenSearch | Kinesis Data Firehose |
| Real-time SQL/Flink analytics on streams | Kinesis Data Analytics |
| Video streaming for ML/analytics | Kinesis Video Streams |

---

## Amazon EventBridge

### Overview
- Serverless **event bus** connecting applications using events
- Successor to CloudWatch Events (with additional capabilities)
- Decouples event producers from consumers
- Near-real-time event delivery; can handle millions of events/second

### EventBridge Components
- **Event bus** — receives events (default bus, custom buses, partner buses)
- **Rules** — match events and route to targets
- **Targets** — Lambda, Step Functions, SQS, SNS, Kinesis, API Gateway, EC2, ECS, etc.

### Key Features
- **Schema Registry** — discover, manage, and share event schemas
- **Archive and Replay** — store events and replay for debugging/testing
- **SaaS Integrations** — natively receive events from Zendesk, Datadog, Shopify, etc.
- **EventBridge Pipes** — point-to-point integrations with optional filtering and enrichment
- **EventBridge Scheduler** — schedule one-time or recurring tasks (replaces CloudWatch Events cron)
- **API Destinations** — invoke external HTTP APIs as targets

> **Exam Tip:** EventBridge = event-driven architecture hub. Use it to decouple microservices. SNS+SQS fan-out is also common. EventBridge supports SaaS integrations that SNS/SQS don't.

---

## AWS Step Functions

### Overview
- Serverless **workflow orchestration** service
- Coordinate distributed applications and microservices
- Define workflows as state machines (Amazon States Language — JSON-based)
- Handles retries, error handling, branching, and parallel execution automatically

### Workflow Types

| Type | Duration | Execution | Audit | Use Case |
|------|----------|-----------|-------|---------|
| **Standard** | Up to 1 year | At-most-once per step | Yes (CloudWatch) | Long-running, auditable workflows |
| **Express** | Up to 5 minutes | At-least-once | CloudWatch Logs | High-volume, short-duration, event-driven |

### Key Features
- Integrates with 200+ AWS services directly (Lambda, DynamoDB, S3, SQS, ECS, etc.)
- **Distributed Map** — process large datasets with parallel iterations (fan-out pattern)
- **Workflow Studio** — visual drag-and-drop workflow design
- Supports synchronous and asynchronous execution
- Monitored via CloudWatch and X-Ray

### Typical Use Cases
- Order processing pipelines
- Data processing workflows (ETL)
- Machine learning training pipelines
- Human approval workflows
- Microservice orchestration

> **Exam Tip:** Step Functions = visual workflow orchestration with built-in error handling, retries, and parallel execution. SWF is the legacy alternative — prefer Step Functions for new architectures.

---

## Amazon SWF (Simple Workflow Service)

### Overview (Legacy)
- Coordinates tasks between application code and human workers
- Task-oriented API; ensures each task assigned exactly once, never duplicated
- Workflow executions can last up to **1 year** (vs. 14 days for SQS)
- Three types of workers: **Actors** (start workflow), **Deciders** (control flow), **Activity Workers** (do the work)
- Legacy service — prefer **Step Functions** for new architectures

> **Exam Tip:** SWF excels at human-in-the-loop workflows (like Amazon warehouse picking). For new architectures, use Step Functions. Key differentiator: SWF supports external signals, has 1-year max duration.

---

## Amazon API Gateway

### Overview
- Fully managed service to **create, publish, maintain, monitor, and secure APIs**
- Serverless frontend for backend services
- No minimum fees; pay only for API calls and data transfer out

### API Types

| Type | Description | Use Case |
|------|-------------|---------|
| **REST API** | Full-featured REST API with API management features | Most use cases |
| **HTTP API** | Lower cost, optimized for Lambda/HTTP backends | Simpler, lower latency |
| **WebSocket API** | Two-way real-time communication | Chat apps, real-time dashboards |
| **gRPC API** | High-performance RPC | Microservices communication |

### Key Features
- **Throttling** — rate limit and burst limit; 429 HTTP response when exceeded
- **Caching** — cache API responses for TTL (reduces backend load); provisioned per stage in GB
- **Stages** — deploy to environments (prod, staging, dev); supports canary deployments
- **Usage Plans** — control access and throttle by API key
- **Lambda proxy integration** — pass entire request to Lambda; Lambda handles all logic
- **AWS WAF integration** — protect APIs from common exploits
- **Private APIs** — accessible only within VPC via Interface Endpoint

### API Gateway CORS
- Cross-Origin Resource Sharing — must enable when client and API are on different domains
- CORS is enforced client-side (browser), not server-side
- If you see "origin policy cannot be read at remote resource" error → enable CORS

### API Gateway + Lambda = Serverless API
- Scale automatically; no infrastructure management
- Event-driven: each API call triggers Lambda invocation

> **Exam Tip:** API Gateway throttling sends HTTP 429. API caching reduces backend hits. Always enable CORS when your JavaScript frontend and API are on different domains.

---

## Amazon AppSync

- Fully managed **GraphQL** API service
- Real-time data subscriptions via WebSockets
- Connects to DynamoDB, Lambda, RDS, HTTP APIs
- Offline data synchronization for mobile apps
- Conflict resolution for concurrent updates

---

## Amazon MQ

### Overview
- Managed **message broker** service for Apache ActiveMQ and RabbitMQ
- Used when **migrating existing on-premises message broker** to cloud (AMQP, MQTT, STOMP, OpenWire)
- No code changes needed — compatible with standard messaging protocols
- Differentiated from SQS: SQS is AWS-native; MQ is for migration of existing apps
- HA: durability-optimized brokers (backed by EFS), throughput-optimized brokers (backed by EBS)

> **Exam Tip:** Use Amazon MQ when migrating existing applications that use JMS/AMQP/MQTT/STOMP messaging. Use SQS/SNS for new AWS-native applications.

---

## Integration Patterns Summary

| Pattern | Services | Use Case |
|---------|----------|---------|
| **Fan-out** | SNS → multiple SQS | Notify multiple systems from one event |
| **Event-driven** | EventBridge → Lambda/Step Functions | Decouple microservices |
| **Request-response** | API Gateway → Lambda | Synchronous API calls |
| **Async decoupling** | SQS between services | Buffer requests, absorb load spikes |
| **Real-time streaming** | Kinesis Data Streams | Continuous data processing |
| **Delivery pipeline** | Kinesis Firehose → S3/Redshift | Stream to storage/analytics |
| **Workflow** | Step Functions | Multi-step business processes |
| **Pub/Sub** | SNS | Push notifications to many subscribers |

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — IAM for API Gateway, SQS/SNS resource policies
- [[AWS-SAA-02-Compute]] — Lambda as event consumers, ECS triggered by SQS
- [[AWS-SAA-03-Storage]] — S3 event notifications to SQS/SNS/Lambda, Firehose to S3
- [[AWS-SAA-04-Databases]] — DynamoDB Streams to Lambda, SQS buffer for DB writes
- [[AWS-SAA-05-Networking-and-CDN]] — API Gateway in VPC, VPC endpoints for SQS/SNS
- [[AWS-SAA-07-Monitoring-and-Management]] — CloudWatch for SQS depth, EventBridge rules
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — Event-driven patterns, decoupling

*AWS SAA-C03/C04 Reference*
