---
title: "AWS SAA — IAM and Security"
created: "2026-04-09"
type: reference-note
status: active
tags:
  - reference-note
  - aws
  - aws-saa
  - certification
---

# AWS SAA — IAM and Security

## IAM (Identity Access Management)

### Overview
- **Global service** — not limited by region; users, groups, roles, and policies are accessible globally
- Centralized hub of control; integrates with all AWS services
- Supports identity federation (delegate auth to Facebook, Google, Active Directory, etc.)
- MFA support; custom password rotation policies
- **PCI DSS compliant** (payment card security regulations)
- Root account = the email used to sign up for AWS — use for billing only, never day-to-day ops

### IAM Entities

| Entity | Description |
|--------|-------------|
| **Users** | Individual end users (employees, CTOs, etc.) |
| **Groups** | Collections of users with shared permissions |
| **Roles** | Permissions assigned to AWS services (e.g., Lambda writing to S3) |
| **Policies** | JSON rule sets attached to identities — grant or limit access |

- **You cannot nest IAM Groups** — individual users can belong to multiple groups, but subgroups are not possible
- New users have **no permissions** by default (least privilege)
- Access key ID + secret access key = programmatic access only (CLI/SDK), not console

### Priority Levels in IAM
1. **Explicit Deny** — highest priority; cannot be overruled by anything
2. **Explicit Allow** — grants access if no explicit deny exists
3. **Default Deny (Implicit Deny)** — all identities start with no access

> **Exam Tip:** An explicit deny ALWAYS wins, even if an allow policy also applies. Default deny means nothing is allowed unless explicitly granted.

### IAM Policy Types
- **Identity-based policies** — attached to users, groups, roles
- **Resource-based policies** — attached to resources (S3 buckets, SQS queues, KMS keys)
- **Service Control Policies (SCPs)** — used with AWS Organizations; restrict what accounts/OUs can do
- **Permission boundaries** — define max permissions an IAM entity can have

### IAM Key Features
- **IAM Roles** can be assigned to EC2 instances at creation or after — change permissions anytime
- **Tags on policies** — control access based on resource tags (e.g., production vs development)
- **Attribute-based access control (ABAC)** — fine-grained permissions using tags
- **IAM Roles Anywhere** — allows workloads outside AWS to use IAM roles

### IAM Security Tools
- **IAM Access Advisor (user level)** — shows services granted to a user and when last accessed; use to revise policies
- **IAM Credentials Report (account level)** — lists all account users and status of credentials

### STS (Security Token Service)
- Creates **temporary security credentials** for trusted users
- Short-term: configurable from minutes to several hours; expire automatically
- Used for identity federation, cross-account access, EC2 instance roles
- `AssumeRole`, `AssumeRoleWithWebIdentity`, `GetFederationToken` are key API calls

> **Exam Tip:** STS is used whenever you need temporary credentials. It is NOT a long-term auth mechanism.

### Identity Federation
- **Web Identity Federation** — authenticate via Facebook/Google/Amazon, receive auth token, exchange for temp AWS credentials via Cognito or STS
- **SAML-based federation** — use Microsoft Active Directory for AWS Management Console login for non-IAM users
- **IAM Identity Center (SSO)** — centralized SSO for multiple AWS accounts and applications

---

## Amazon Cognito

### Cognito User Pools vs Identity Pools

| Feature | User Pools | Identity Pools |
|---------|-----------|----------------|
| Purpose | Sign-up / sign-in directory | Grant temporary AWS credentials |
| Output | JWT tokens | IAM role credentials |
| Use case | App authentication | Direct AWS service access (S3, DynamoDB) |

- **User Pools** — user directories for registration, authentication, password recovery; issues JWT tokens
- **Identity Pools (Federated Identities)** — grant users temp AWS credentials to access services directly
- Supports: username/password, social login (Facebook, Google, Apple, Amazon), MFA (SMS/TOTP), SAML, OIDC
- Integrates with API Gateway and Lambda for custom auth
- **Cognito's job**: broker between your app and legitimate authenticators

> **Exam Tip:** User Pools = authentication (who are you?). Identity Pools = authorization (what can you do in AWS?). You often chain them together.

---

## AWS KMS (Key Management Service)

### Overview
- Managed service for creating and controlling **cryptographic keys**
- Keys protected by **FIPS 140-2 validated** hardware security modules (HSMs)
- Integrates with S3, RDS, EBS, DynamoDB, Lambda, CloudTrail, etc.
- Encrypts data **up to 4KB per API call** — for larger data, use **envelope encryption** (KMS encrypts the data key, data key encrypts the data)

### Key Types
- **Symmetric keys** — AES-256, same key for encrypt/decrypt (most common)
- **Asymmetric keys** — RSA/ECC key pairs; public key can be used outside AWS
- **HMAC keys** — for message authentication

### Key Features
- **Automatic key rotation** — annual rotation for AWS-managed keys; configurable for customer-managed keys
- **Custom key stores** — backed by CloudHSM or external key managers
- **Key auditing** — all key operations logged in CloudTrail
- Multi-region keys — replicate keys across regions

> **Exam Tip:** KMS can only encrypt/decrypt up to 4KB directly. For larger payloads (like S3 objects), it uses envelope encryption — KMS encrypts a data key, which is then used to encrypt the actual data.

---

## AWS WAF (Web Application Firewall)

### Overview
- **Layer 7 firewall** — operates on HTTP/HTTPS, URL query strings, headers, body
- Integrates with: CloudFront, API Gateway, ALB, AppSync
- Protects against OWASP Top 10 (SQL injection, XSS, etc.)

### WAF Behaviors
- **Allow all except specified** — whitelist approach
- **Block all except specified** — restrict to known users
- **Count matching requests** — test rules before blocking

### WAF Protection Capabilities
- IP address filtering (by IP or CIDR range)
- Country of origin filtering
- Request header inspection
- String matching (exact or regex)
- Request size constraints
- **SQL injection detection**
- **Cross-site scripting (XSS) detection**

> **Exam Tip:** Use NACLs to block malicious IPs at the network level, WAF for application-level (Layer 7) filtering. Defense in depth = use both.

---

## AWS Shield

### Shield Standard
- **Free, automatically enabled** for all AWS customers
- Protects against common, most frequent DDoS attacks (Layer 3/4)

### Shield Advanced
- **Paid service** — $3,000/month per organization
- Enhanced DDoS protection for EC2, ELB, CloudFront, Global Accelerator, Route 53
- 24/7 access to AWS DDoS Response Team (DRT)
- Cost protection (credits for DDoS-related scaling)
- Real-time attack visibility and forensic reports

> **Exam Tip:** Shield Standard = free, automatic, basic. Shield Advanced = paid, enterprise-grade, with DRT access.

---

## Amazon GuardDuty

- **Threat detection service** using machine learning
- Analyzes: CloudTrail logs, VPC Flow Logs, DNS logs, S3 data events, EKS audit logs, RDS login activity
- Detects: unauthorized access, malware, data exfiltration, cryptojacking, unusual API calls
- Generates **findings** for review; integrates with EventBridge for automated response
- **Regional service** — findings can be aggregated via Security Hub
- Supports multi-account management through AWS Organizations
- **GuardDuty Malware Protection** — scans EC2 instances and containers for malware
- No infrastructure to manage — fully managed

> **Exam Tip:** GuardDuty = intelligent threat detection. It does NOT prevent attacks — it detects them and generates findings. Pair with EventBridge + Lambda for automated remediation.

---

## Amazon Inspector

- **Automated security assessment** for EC2 instances and ECR container images
- Scans for: software vulnerabilities (CVEs), unintended network exposure, security best practice deviations
- **Agent-based** for EC2, **agentless** for container images in ECR
- Continuous scanning; prioritizes findings by severity
- Integrates with Systems Manager for agent deployment
- Available across all AWS Regions; multi-account via Organizations

> **Exam Tip:** Inspector = vulnerability scanning for EC2 and containers. GuardDuty = threat detection at the account/network level. They're complementary, not substitutes.

---

## Amazon Macie

- **Data security service** using ML to discover and classify sensitive data in **S3**
- Detects: PII (SSNs, credit card numbers, passport numbers), financial data, intellectual property, healthcare data
- Provides: data sensitivity scores per S3 bucket, automated findings, alerts
- Can analyze CloudTrail logs for access to sensitive data
- Integrates with EventBridge and SNS for automated response
- Multi-account management via Organizations
- Dashboard for centralized S3 data security posture

> **Exam Tip:** Macie is specifically for S3 data classification and PII detection. If the question involves finding sensitive data in S3, the answer is Macie.

---

## AWS Security Hub

- **Centralized security posture management** across AWS accounts and regions
- Aggregates findings from: GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, and third-party tools
- **Compliance Standards supported**: PCI DSS v4.0, AWS FSBP, CIS Benchmarks, NIST SP 800-53, ISO 27001
- **Security Score** — quantitative measure of compliance status
- Cross-region and cross-account aggregation
- Integrates with AWS Organizations
- Automated remediation via Systems Manager Automation and Lambda
- **Regional service** — must enable in each region

> **Exam Tip:** Security Hub = single pane of glass for security findings across your entire AWS org. It aggregates, not generates findings (except its own compliance checks).

---

## IAM Access Analyzer

- Analyzes IAM policies to identify **unintended external access**
- Identifies: excessive permissions, unintended resource access, public access
- Analyzes: S3 buckets, KMS keys, IAM roles, SQS queues, Lambda functions, Secrets Manager secrets
- Generates findings for resources accessible from outside your AWS account
- Available across all Regions; multi-account via Organizations
- **Zone of trust** = your account or organization; anything outside triggers a finding

> **Exam Tip:** IAM Access Analyzer finds policies that allow external access you didn't intend. Different from Access Advisor (which shows what's been used).

---

## AWS Trusted Advisor — Security Pillar

Key security checks (some free, some Business/Enterprise support only):
- MFA on root account
- IAM use (not just root)
- Public S3 bucket access
- Security group: specific ports unrestricted
- EBS/RDS public snapshots
- Exposed access keys
- CloudTrail logging enabled

---

## Key Comparisons

### Security Services at a Glance

| Service | Purpose | Scope |
|---------|---------|-------|
| **WAF** | Block web attacks (Layer 7) | CloudFront, ALB, API GW |
| **Shield** | DDoS protection | AWS infrastructure |
| **GuardDuty** | Threat detection (ML) | Account-wide |
| **Inspector** | Vulnerability scanning | EC2, ECR containers |
| **Macie** | PII discovery in S3 | S3 buckets |
| **Security Hub** | Aggregated findings | All accounts/regions |
| **IAM Access Analyzer** | Policy analysis | IAM policies |
| **Config** | Compliance/config tracking | Resource configs |

---

## Connections

- [[AWS-SAA-02-Compute]] — IAM roles for EC2, Lambda execution roles
- [[AWS-SAA-03-Storage]] — S3 bucket policies, SSE-KMS, Macie for S3
- [[AWS-SAA-04-Databases]] — RDS IAM authentication, Secrets Manager
- [[AWS-SAA-05-Networking-and-CDN]] — WAF with CloudFront/ALB, VPC security groups, NACLs
- [[AWS-SAA-07-Monitoring-and-Management]] — CloudTrail for audit, Config for compliance, Systems Manager
- [[AWS-SAA-08-Well-Architected-and-Architecture-Patterns]] — Security pillar of Well-Architected Framework

*AWS SAA-C03/C04 Reference*
