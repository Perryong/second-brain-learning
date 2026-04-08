---
title: "AWS SAA — Monitoring and Management"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — Monitoring and Management

## Amazon CloudWatch

### Overview
- Monitoring and observability service for AWS resources and applications
- Collects: **metrics**, **logs**, **events**
- CloudWatch is about **performance**; CloudTrail is about **auditing** — do not confuse them

### CloudWatch Metrics
- Time-ordered set of data points (variables monitored over time)
- **Default interval**: every 5 minutes for EC2
- **Detailed Monitoring**: 1-minute intervals (additional cost; enable per instance)
- **High resolution custom metrics**: sub-minute intervals down to 1 second
- EC2 **host-level metrics** available by default: CPU, network, disk, status checks
- EC2 metrics **NOT collected by default** (require CloudWatch agent or custom metrics):
  - Memory utilization
  - Disk swap utilization
  - Disk space utilization
  - Page file utilization
  - Log collection

> **Exam Tip:** Memory utilization is NOT a default CloudWatch metric for EC2 — it requires the CloudWatch Agent or a custom metric. This is a common exam gotcha.

### CloudWatch Alarms
- Monitor metrics and trigger actions when thresholds are crossed
- States: `OK`, `ALARM`, `INSUFFICIENT_DATA`
- Actions: SNS notification, Auto Scaling action, EC2 action (stop/terminate/reboot/recover)
- Example: billing alarm when monthly charges exceed $X

### CloudWatch Logs
- Centralize logs from EC2, Lambda, CloudTrail, Route 53, on-premises servers
- **Log Groups** — logical grouping of log streams
- **Log Streams** — sequence of log events from single source
- **CloudWatch Logs Insights** — interactive query and analysis of log data (like SQL for logs)
- **Subscription Filters** — stream log data in real-time to Lambda, Kinesis, Firehose

### CloudWatch Agent
- Multi-platform agent (Linux and Windows)
- Enables: custom metrics, memory metrics, disk metrics, log file collection
- Works for EC2 instances AND on-premises servers
- Replaces separate SSM agent for metrics

### CloudWatch Dashboards
- Customizable home pages for monitoring resources in a single view
- Cross-region dashboards possible
- Integrate metrics and alarms

### CloudWatch Events / EventBridge
- Near-real-time stream of system events for resource state changes
- Trigger Lambda, SNS, SQS, Step Functions based on events
- Now part of Amazon EventBridge (see Messaging and Integration note)

### CloudWatch Container Insights
- Collect, aggregate, and summarize metrics and logs from containerized applications
- Works with ECS, EKS, Kubernetes on EC2, Fargate

### CloudWatch Application Signals
- Application performance monitoring (APM)
- Automatically instrument applications to collect performance data

### CloudWatch Synthetics
- Create **canaries** — configurable scripts that monitor endpoints and APIs
- Checks availability and latency; stores screenshots, HTTP archive files
- Runs on schedule; detects issues before users report them

### CloudWatch Contributor Insights
- Analyze log data and find top contributors to performance issues
- Helps identify top talkers, heavy hitters, and root causes

### CloudWatch Internet Monitor
- Monitor internet performance for your applications
- Identifies performance and availability issues from internet routing

---

## AWS CloudTrail

### Overview
- Service for **governance, compliance, operational auditing, and risk auditing**
- Logs all **API calls** made to your AWS account (console, CLI, SDK, other services)
- Think: "Who did what, when, from where in my AWS account?"
- **Regional service** — configure to collect trails in all regions
- Event history stored **90 days by default** (free; no trail needed)

### CloudTrail Trails
- Configure trails to persist events beyond 90 days
- Logs stored in **S3**; optionally stream to CloudWatch Logs
- Log files **encrypted by default** with SSE-S3; optionally KMS
- S3 lifecycle rules can archive or delete old logs
- SNS notifications for log delivery and validation
- Logs can be retained up to **2 years** (using CloudTrail Lake)

### Event Types

| Type | Description | Default Logged? |
|------|-------------|----------------|
| **Management Events** | Operations on resources (create, delete, modify); console actions, policy changes, login | **Yes** |
| **Data Events** | Resource operations (S3 object-level API, Lambda invocations) | **No** (higher volume, must enable) |
| **Insights Events** | Anomalous API activity detection | No (must enable) |

> **Exam Tip:** CloudTrail logs management events by default. Data events (S3 GetObject, Lambda invocations) are NOT logged by default — you must explicitly enable them. This costs more.

### CloudTrail Lake
- Immutable, managed data lake for CloudTrail events
- SQL-based querying (like Athena) without needing S3/Athena setup
- Store events for up to **7 years**

### CloudTrail Insights
- Automatically detect unusual API activity (anomalies in write management events)
- Compares current activity to baseline; generates findings when anomalies detected

### CloudTrail Integration
- **Athena** — query CloudTrail logs in S3 using SQL (serverless analysis)
- **CloudWatch Logs** — set alarms on specific API activity
- **GuardDuty** — analyzes CloudTrail logs for threat detection
- **Security Hub** — aggregates findings

---

## AWS Config

### Overview
- Continuously monitors and records AWS **resource configurations**
- Evaluates configs against desired/compliance rules
- Historical record of configuration changes
- NOT a real-time prevention service — it records and evaluates

### Key Capabilities
- Evaluate resource configurations for desired settings
- Snapshot current configurations of all supported resources
- Retrieve historical configurations
- Receive notifications when resources are created, modified, or deleted
- View relationships between resources (e.g., what security group is this EC2 using?)

### Config Rules
- **Managed rules** — pre-built by AWS (e.g., "are all EBS volumes encrypted?")
- **Custom rules** — use Lambda for custom compliance logic
- Rules run on: configuration change, periodic schedule, or both

### Config Remediation
- **Automatic remediation** — Systems Manager Automation document triggered on non-compliance
- **Manual remediation** — view findings and take action yourself

### Config Aggregators
- Aggregate Config data across multiple accounts and regions into single view
- Useful for multi-account governance

> **Exam Tip:** AWS Config answers "Is my infrastructure compliant with my policies?" It does NOT prevent changes — it detects and records them. For prevention, use IAM/SCPs.

---

## AWS CloudFormation

### Overview
- Infrastructure as Code (IaC) — provision AWS resources using templates
- Templates in **YAML or JSON**
- A full CloudFormation setup = a **Stack**
- One template can create unlimited stacks
- Provisions infrastructure consistently and repeatably

### Template Structure
- **Resources** — only mandatory field (defines what to create)
- Parameters — input values for templates
- Mappings — conditional values based on region/environment
- Outputs — values to export from stack
- Conditions — control resource creation based on conditions

### Key Features
- **StackSets** — deploy CloudFormation stacks across **multiple accounts and regions**
- **ChangeSets** — preview changes before applying (like a dry-run diff)
- **Drift Detection** — identify when deployed stack differs from template
- **Rollback triggers** — monitor stack creation; rollback on error
- **CloudFormation Guard** — policy-as-code validation for templates
- **Resource Import** — bring existing resources under CloudFormation management
- **Nested Stacks** — complex deployments with modular, reusable stack templates
- **Stack Policies** — protect stack resources from unintentional updates/deletes

> **Exam Tip:** CloudFormation = Infrastructure as Code (IaC). Elastic Beanstalk uses CloudFormation under the hood but hides the complexity. CDK generates CloudFormation templates.

---

## AWS Systems Manager (SSM)

### Overview
- Centralized operations management for AWS and on-premises resources
- Manage and automate operational tasks at scale

### Key Capabilities

#### SSM Parameter Store
- Secure, hierarchical storage for **configuration data and secrets**
- Tiers: Standard (free, up to 4KB, no rotation), Advanced (larger values, higher throughput)
- Supports: plain text, SecureString (KMS-encrypted)
- Version tracking, parameter history
- Integration with: Lambda, EC2, ECS, CloudFormation

#### Secrets Manager vs Parameter Store

| Feature | Secrets Manager | Parameter Store |
|---------|----------------|-----------------|
| Automatic rotation | **Yes** (native) | No (manual via Lambda) |
| Cost | Per secret/month | Free (Standard tier) |
| Cross-account access | Yes | Yes |
| Primary use | DB credentials, API keys | Config data, non-secret values |
| Integration with RDS | Native rotation support | Manual |

> **Exam Tip:** Secrets Manager = automatic rotation of secrets (especially DB passwords). Parameter Store = cheaper, good for non-secret config values. For DB credentials that need auto-rotation, use Secrets Manager.

#### SSM Session Manager
- **Browser-based shell** or AWS CLI access to EC2 instances
- **No need for SSH, bastion hosts, or open inbound ports**
- Full audit trail in CloudTrail; logs sessions to S3 or CloudWatch
- Works with instances that have SSM Agent installed
- Supports Windows (PowerShell) and Linux (bash)

#### SSM Patch Manager
- Automate patching of OS and applications across your fleet
- Define patch baselines; schedule maintenance windows
- Patch compliance reporting
- Works on-premises and in AWS

#### SSM Run Command
- Execute commands on managed instances remotely without SSH
- Run shell scripts, PowerShell commands
- Results visible in console or stored in S3/CloudWatch Logs

#### SSM State Manager
- Maintain defined state for EC2 instances (e.g., always have antivirus installed)
- Associate documents with instances; enforce state on schedule

#### SSM Inventory
- Collect software inventory from managed instances
- Track: installed applications, network config, OS version, etc.

---

## AWS Trusted Advisor

- Real-time best practice guidance across 5 categories:
  1. **Cost Optimization** — identify underutilized resources, savings opportunities
  2. **Performance** — improve speed and responsiveness
  3. **Security** — close security gaps, audit permissions
  4. **Fault Tolerance** — increase resilience
  5. **Service Limits** — check if approaching AWS service limits

- **Basic/Developer support**: 6 free security checks + 4 performance checks
- **Business/Enterprise support**: full Trusted Advisor checks + AWS Support API
- Use with **EventBridge** to automate responses to Trusted Advisor findings

---

## AWS Cost Explorer

- Visualize, understand, and manage AWS costs and usage
- Pre-built reports and custom queries
- Time range: up to 13 months of historical data + 12 months of forecasting
- Filter by service, account, tag, region, etc.
- **Savings Plans recommendations**
- **Rightsizing recommendations** (identify oversized EC2 instances)
- Reserved Instance utilization and coverage reports

---

## AWS Budgets

- Set custom budgets and get alerts when costs or usage exceed thresholds
- Budget types: Cost, Usage, Savings Plans utilization/coverage, Reserved Instance
- Alert via SNS or email; up to 5 SNS notifications per budget
- Can trigger automated actions (e.g., apply IAM policy to restrict spending)

---

## AWS Organizations

### Overview
- Account management service for consolidating multiple AWS accounts
- **Best practice**: root account for billing only; separate accounts for resources
- **Consolidated billing** — single payment method, volume discounts

### Organizational Units (OUs)
- Group similar accounts into OUs
- Attach policy to OU — all accounts in OU inherit it
- Hierarchical structure (master account → OUs → member accounts)

### Service Control Policies (SCPs)
- Policy-based controls applied to accounts or OUs
- Define **maximum permissions** — even if IAM allows it, SCP can restrict it
- SCPs do NOT grant permissions — they restrict what IAM can allow
- Use to: restrict regions, prevent resource deletion, enforce tagging

> **Exam Tip:** SCPs are guardrails. They set the maximum permissions any IAM policy can grant in an account. An SCP allowing an action doesn't grant it — IAM must also allow it.

### AWS Control Tower

- Automate set up of a **multi-account landing zone** following AWS best practices
- Manages account provisioning, SCPs, logging, and monitoring
- Built on top of AWS Organizations and AWS SSO
- **Account Factory** — provision new accounts with pre-defined configs
- **Guardrails** — preventive (SCPs) or detective (Config rules) controls
- **Dashboard** — visibility into compliance across all accounts

---

## Summary: Which Service for What

| Question | Service |
|----------|---------|
| How is my app performing? | CloudWatch Metrics/Alarms |
| What's in my logs? | CloudWatch Logs / Logs Insights |
| Who made API calls? | CloudTrail |
| Is my infra compliant? | AWS Config |
| Deploy infra as code | CloudFormation |
| Access EC2 without SSH | SSM Session Manager |
| Rotate DB credentials | Secrets Manager |
| Store config values | SSM Parameter Store |
| Patch EC2 fleet | SSM Patch Manager |
| Get cost recommendations | Trusted Advisor / Cost Explorer |
| Set spending alerts | AWS Budgets |
| Manage multiple accounts | AWS Organizations / Control Tower |

---

## Connections

- [[AWS-SAA-01-IAM-and-Security]] — CloudTrail for audit, Config compliance, Security Hub
- [[AWS-SAA-02-Compute]] — CloudWatch for EC2 metrics, SSM for patch/access, Auto Scaling alarms
- [[AWS-SAA-03-Storage]] — S3 for CloudTrail logs, Config for S3 compliance
- [[AWS-SAA-04-Databases]] — RDS Enhanced Monitoring, Secrets Manager for DB creds
- [[AWS-SAA-05-Networking-and-CDN]] — VPC Flow Logs to CloudWatch, WAF monitoring
- [[AWS-SAA-06-Messaging-and-Integration]] — EventBridge from CloudWatch Events, SQS depth alarms
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — Operational Excellence pillar

*AWS SAA-C03/C04 Reference*
