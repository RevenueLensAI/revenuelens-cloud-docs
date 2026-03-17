# Security Hub Failed Controls - Dev Environment

**Account:** 816671155797 | **Region:** us-west-2 | **Last reviewed:** 2026-03-17

This document tracks the rationale for each suppressed or disabled control
so that decisions can be revisited without losing context.

---

## Category 1: Architecturally Irrelevant — DISABLED

These 14 controls were **disabled** on 2026-03-17 because they flag services or patterns
we do not use. Disabling removes them from the security score and controls page.
**Re-evaluate if** the architecture changes to include these services.
To re-enable: `aws securityhub batch-update-standards-control-associations --standards-control-association-updates '[{"StandardsArn":"arn:aws:securityhub:us-west-2::standards/aws-foundational-security-best-practices/v/1.0.0","SecurityControlId":"<CONTROL_ID>","AssociationStatus":"ENABLED"}]' --profile revenuelens-dev`

### GuardDuty / Inspector - Services We Don't Use

| Control | Title | Severity | Why Suppressed |
|---------|-------|----------|----------------|
| GuardDuty.6 | GuardDuty Lambda Protection should be enabled | HIGH | We don't use Lambda. No Lambda functions to protect. |
| GuardDuty.7 | GuardDuty EKS Runtime Monitoring should be enabled | HIGH | We don't use EKS. EC2+Docker architecture. |
| GuardDuty.12 | GuardDuty ECS Runtime Monitoring should be enabled | MEDIUM | We don't use ECS/Fargate. Docker Compose on EC2. |
| Inspector.2 | Amazon Inspector ECR scanning should be enabled | HIGH | Not using Inspector. ECR repos are internal build artifacts only. |
| Inspector.3 | Amazon Inspector Lambda code scanning should be enabled | HIGH | We don't use Lambda. |
| Inspector.4 | Amazon Inspector Lambda standard scanning should be enabled | HIGH | We don't use Lambda. |

### VPC Endpoints - Not Cost-Justified for Dev

| Control | Title | Severity | Why Suppressed |
|---------|-------|----------|----------------|
| EC2.10 | VPC should have endpoint for EC2 service | MEDIUM | VPC endpoints cost ~$7/mo each. Dev traffic is minimal and goes over IGW. |
| EC2.55 | VPC should have endpoint for ECR API | MEDIUM | Same — not cost-justified in dev. |
| EC2.56 | VPC should have endpoint for Docker Registry | MEDIUM | Same — not cost-justified in dev. |
| EC2.57 | VPC should have endpoint for Systems Manager | MEDIUM | Same — not cost-justified in dev. |
| EC2.58 | VPC should have endpoint for SSM Incident Manager Contacts | MEDIUM | Same — not using Incident Manager. |
| EC2.60 | VPC should have endpoint for SSM Incident Manager | MEDIUM | Same — not using Incident Manager. |

### Other Inapplicable Controls

| Control | Title | Severity | Why Suppressed |
|---------|-------|----------|----------------|
| Macie.1 | Macie should be enabled | MEDIUM | Macie costs ~$1/GB scanned. No sensitive data classification needs in dev. Re-evaluate for prod if handling PII. |
| EC2.172 | VPC Block Public Access should block IGW traffic | MEDIUM | Our architecture requires internet access via IGW for the ALB and EC2 instance. Blocking IGW would break the application. |

---

## Category 2: Accepted Risk - Dev Environment Only

Controls where we've consciously accepted the risk for the dev environment.
**Must be addressed before prod parity or compliance audits.**

### Public Network Exposure

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| EC2.9 | EC2 instances should not have a public IPv4 address | HIGH | Dev instance needs public IP for ALB and SSH access. Prod should use private subnets + NAT gateway. | Prod hardening |
| EC2.15 | Subnets should not auto-assign public IPs | MEDIUM | Public subnets are part of current architecture. 2 subnets affected. | Prod hardening — move to private subnets with NAT |
| RDS.46 | RDS should not be in public subnets with IGW routes | HIGH | RDS is in public subnets but security groups restrict access to VPC only. Functional risk is low, but should use private subnets in prod. | Prod hardening |

### RDS Configuration

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| RDS.5 | RDS should be configured with multiple AZs | MEDIUM | Multi-AZ doubles cost. Acceptable for dev where downtime is tolerable. | Prod (already should be multi-AZ) |
| RDS.9 | RDS should publish logs to CloudWatch | MEDIUM | Adds cost for CloudWatch log storage. Not critical for dev debugging. | Prod — enable postgresql and upgrade log exports |
| RDS.36 | RDS PostgreSQL should publish logs to CloudWatch | MEDIUM | Same as RDS.9 — duplicate control for PostgreSQL specifically. | Prod |
| RDS.10 | IAM authentication should be configured for RDS | MEDIUM | Using Secrets Manager for credentials instead. IAM auth adds complexity for marginal benefit. | If moving to IAM-based access patterns |
| RDS.6 | Enhanced monitoring should be configured for RDS | LOW | Adds ~$3/mo. Not needed for dev where CloudWatch basic metrics suffice. | Prod — enable at 60s granularity |
| RDS.23 | RDS should not use default port (5432) | LOW | Changing the port is security-through-obscurity. Actual access is controlled by security groups. Accepted risk. | Never — low value control |

### Secrets Management

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| SecretsManager.1 | Secrets should have automatic rotation enabled | MEDIUM | Rotation requires Lambda functions and application support for credential refresh. 5 secrets affected: SESSION_SECRET, developer_pem, deployer_pem, db_master, ubuntu_pro_token. | When app supports graceful credential rotation |
| SecretsManager.4 | Secrets should be rotated within 90 days | MEDIUM | Same root cause as SecretsManager.1 — no rotation configured. 4 secrets affected. | Same as above |

### Encryption In Transit (ALB to EC2)

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| ELB.21 | Target group health checks should use HTTPS | MEDIUM | ALB terminates TLS. Internal traffic (ALB → EC2) is HTTP on a private network. 7 target groups affected. | If compliance requires end-to-end encryption |
| ELB.22 | Target groups should use encrypted transport | MEDIUM | Same as ELB.21 — ALB-to-EC2 traffic is unencrypted HTTP. Standard pattern for TLS termination at ALB. 7 target groups. | If compliance requires end-to-end encryption |

### ECR Configuration

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| ECR.2 | ECR repos should have tag immutability | MEDIUM | We re-push the `latest` tag during development. Immutability would break this workflow. 10 repos affected. | If moving to immutable deployment tags (e.g., git SHA) |

### KMS / IAM

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| KMS.2 | IAM principals should not allow KMS decrypt on all keys | MEDIUM | terraform-ops IAM group has broad KMS access for Terraform operations. Could scope down to specific key ARNs. | Next IAM policy review |

### DynamoDB

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| DynamoDB.2 | DynamoDB tables should have PITR enabled | MEDIUM | Only table is terraform-locks. PITR costs extra and the table is trivially recreatable. | Never — low value for lock table |

---

## Category 3: Logging & Monitoring Gaps

Controls related to CloudWatch log metric filters and S3 access logging.
These are best-practice monitoring that adds cost and operational overhead.

### S3 Access Logging

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| S3.9 | S3 buckets should have access logging enabled | MEDIUM | 6 buckets affected. Access logging creates additional log data and requires a destination bucket. Low value in dev. | Prod — enable for frontend and terraform state buckets at minimum |

### CloudTrail

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| CloudTrail.5 | CloudTrail should integrate with CloudWatch Logs | MEDIUM | Sending CloudTrail to CloudWatch adds ~$0.50/GB ingestion cost. CloudTrail events are still in S3. | Prod — needed for the CloudWatch metric filters below |
| CloudTrail.7 | CloudTrail S3 bucket should have access logging | LOW | Logging bucket for logging bucket — diminishing returns in dev. | Prod if required by compliance |

### CloudWatch Log Metric Filters (CIS Benchmark)

All of these require CloudTrail → CloudWatch integration (CloudTrail.5) as a prerequisite.
They are part of the CIS AWS Foundations Benchmark and create metric filters + alarms
for specific API activity patterns. **Enabling CloudTrail.5 is the prerequisite for all of these.**

| Control | Title | Severity |
|---------|-------|----------|
| CloudWatch.1 | Log metric filter for root user usage | LOW |
| CloudWatch.2 | Log metric filter for unauthorized API calls | LOW |
| CloudWatch.3 | Log metric filter for console sign-in without MFA | LOW |
| CloudWatch.4 | Log metric filter for IAM policy changes | LOW |
| CloudWatch.5 | Log metric filter for CloudTrail config changes | LOW |
| CloudWatch.6 | Log metric filter for console auth failures | LOW |
| CloudWatch.7 | Log metric filter for CMK disable/deletion | LOW |
| CloudWatch.8 | Log metric filter for S3 bucket policy changes | LOW |
| CloudWatch.9 | Log metric filter for AWS Config changes | LOW |
| CloudWatch.10 | Log metric filter for security group changes | LOW |
| CloudWatch.11 | Log metric filter for NACL changes | LOW |
| CloudWatch.12 | Log metric filter for network gateway changes | LOW |
| CloudWatch.13 | Log metric filter for route table changes | LOW |
| CloudWatch.14 | Log metric filter for VPC changes | LOW |

**To address all 14 CloudWatch controls:** Enable CloudTrail.5 first (~$0.50/GB/mo),
then create the metric filters and SNS alarms (~$0.10/alarm/mo each). Total incremental
cost depends on CloudTrail volume but likely $2-5/mo for dev.

### SSM

| Control | Title | Severity | Why Suppressed | Revisit When |
|---------|-------|----------|----------------|--------------|
| SSM.6 | SSM Automation should have CloudWatch logging | MEDIUM | Not actively using SSM Automation documents. | If we start using SSM Automation runbooks |

---

## Summary

| Category | Controls | Highest Severity |
|----------|----------|-----------------|
| Architecturally irrelevant (DISABLED) | 14 | HIGH (Lambda/EKS — services not in use) |
| Accepted risk (dev) | 16 | HIGH (public IP, RDS subnet) |
| Logging & monitoring gaps | 18 | MEDIUM |
| **Total** | **48** | |

### Priority Remediation for Prod

If/when hardening for production, address in this order:

1. **Network isolation** (EC2.9, EC2.15, RDS.46) — move to private subnets with NAT gateway
2. **RDS hardening** (RDS.5, RDS.9, RDS.36, RDS.6) — multi-AZ, logging, enhanced monitoring
3. **CloudTrail → CloudWatch** (CloudTrail.5) — unlocks all 14 CIS metric filter controls
4. **Secrets rotation** (SecretsManager.1, SecretsManager.4) — requires app-level changes
5. **S3 access logging** (S3.9) — enable for key buckets
6. **KMS policy scoping** (KMS.2) — restrict terraform-ops to specific key ARNs
