# Security Hub Failed Controls - Prod Environment

**Account:** 500532068543 | **Region:** us-west-2 | **Last reviewed:** 2026-03-17

This document tracks the rationale for each suppressed or disabled control
so that decisions can be revisited without losing context.

---

## Already Disabled Controls

These 11 controls were disabled prior to this review (date unknown).

| Control | Title | Severity | Likely Reason |
|---------|-------|----------|---------------|
| GuardDuty.7 | GuardDuty EKS Runtime Monitoring | HIGH | No EKS usage |
| EC2.10 | VPC endpoint for EC2 service | MEDIUM | Cost — VPC endpoints ~$7/mo each |
| EC2.55 | VPC endpoint for ECR API | MEDIUM | Cost |
| EC2.58 | VPC endpoint for SSM Incident Manager Contacts | MEDIUM | Not using Incident Manager |
| EC2.60 | VPC endpoint for SSM Incident Manager | MEDIUM | Not using Incident Manager |
| Macie.1 | Macie should be enabled | MEDIUM | Cost — no PII classification needs |
| SecretsManager.1 | Secrets should have automatic rotation | MEDIUM | App doesn't support graceful credential rotation |
| SecretsManager.4 | Secrets should be rotated within 90 days | MEDIUM | Same as SecretsManager.1 |
| RDS.10 | IAM authentication for RDS | MEDIUM | Using Secrets Manager for credentials instead |
| EC2.15 | Subnets should not auto-assign public IPs | MEDIUM | Public subnets are part of current architecture |
| DynamoDB.2 | DynamoDB PITR | MEDIUM | Only table is terraform-locks — trivially recreatable |

---

## Category 1: Architecturally Irrelevant — DISABLED

These 8 controls were **disabled** on 2026-03-17 because they flag services or patterns
we do not use. Disabling removes them from the security score and controls page.
**Re-evaluate if** the architecture changes to include these services.
To re-enable: `aws securityhub batch-update-standards-control-associations --standards-control-association-updates '[{"StandardsArn":"arn:aws:securityhub:us-west-2::standards/aws-foundational-security-best-practices/v/1.0.0","SecurityControlId":"<CONTROL_ID>","AssociationStatus":"ENABLED"}]' --profile revenuelens-prod`

### Services We Don't Use

| Control | Title | Severity | Why Irrelevant | Status |
|---------|-------|----------|----------------|--------|
| GuardDuty.12 | GuardDuty ECS Runtime Monitoring should be enabled | MEDIUM | We don't use ECS/Fargate. Docker Compose on EC2. | Suppressed |
| Inspector.2 | Amazon Inspector ECR scanning should be enabled | HIGH | Not using Inspector. Could be relevant since we have ECR repos. | Suppressed |
| Inspector.3 | Amazon Inspector Lambda code scanning should be enabled | HIGH | We don't use Lambda. | Suppressed |
| Inspector.4 | Amazon Inspector Lambda standard scanning should be enabled | HIGH | We don't use Lambda. | Suppressed |

### Incompatible with Architecture

| Control | Title | Severity | Why Irrelevant | Status |
|---------|-------|----------|----------------|--------|
| EC2.172 | VPC Block Public Access should block IGW traffic | MEDIUM | Architecture requires IGW for ALB and EC2 internet access. | Suppressed |
| EC2.56 | VPC endpoint for Docker Registry | MEDIUM | Cost — ~$7/mo. Traffic goes over IGW. | Suppressed |
| EC2.57 | VPC endpoint for Systems Manager | MEDIUM | Cost — ~$7/mo. SSM traffic goes over IGW. | Suppressed |
| SSM.6 | SSM Automation should have CloudWatch logging | MEDIUM | Not using SSM Automation runbooks. | Suppressed |

**Total: 8 controls — DISABLED 2026-03-17**

---

## Category 2: Accepted Risk — Should Address

Controls representing real security gaps in production. These warrant remediation planning.

### CRITICAL

| Control | Title | Resources | Why Suppressed | Recommendation |
|---------|-------|-----------|----------------|----------------|
| IAM.6 | Hardware MFA should be enabled for root user | Account root | Virtual MFA is enabled but hardware MFA (YubiKey etc.) is not. | **Address this.** Purchase a hardware MFA device. This is the single most impactful security improvement available. ~$25 one-time cost. |

### Network Exposure

| Control | Title | Severity | Resources | Why Suppressed | Recommendation |
|---------|-------|----------|-----------|----------------|----------------|
| EC2.9 | EC2 should not have public IPv4 address | HIGH | i-0a8b752bcdd1bc1a5 | Instance needs public IP for ALB and SSH access. | Move to private subnets + NAT gateway. Significant architecture change. |
| RDS.46 | RDS should not be in public subnets with IGW routes | HIGH | revenuelens-pg | RDS is in public subnets but SGs restrict to VPC only. | Move RDS to private subnets. Should be done alongside EC2.9. |

### Database

| Control | Title | Severity | Resources | Why Suppressed | Recommendation |
|---------|-------|----------|-----------|----------------|----------------|
| RDS.5 | RDS should be multi-AZ | MEDIUM | revenuelens-pg | Doubles cost (~$50/mo extra). | **Enable for prod.** Downtime during failover without multi-AZ is a real business risk. |
| RDS.6 | Enhanced monitoring for RDS | LOW | revenuelens-pg | Adds ~$3/mo. | **Enable for prod.** Low cost, high value for debugging performance issues. 60s granularity recommended. |
| RDS.23 | RDS should not use default port | LOW | revenuelens-pg | Security-through-obscurity. SGs control access. | Low value. Keep suppressed. |

### ALB-to-EC2 Encryption

| Control | Title | Severity | Findings | Why Suppressed | Recommendation |
|---------|-------|----------|----------|----------------|----------------|
| ELB.21 | Target group health checks should use HTTPS | MEDIUM | 6 target groups | ALB terminates TLS. Internal ALB→EC2 traffic is HTTP on private network. | Acceptable if VPC is trusted. Address if compliance requires e2e encryption. |
| ELB.22 | Target groups should use encrypted transport | MEDIUM | 6 target groups | Same as ELB.21. | Same. |

### IAM / KMS

| Control | Title | Severity | Resources | Why Suppressed | Recommendation |
|---------|-------|----------|-----------|----------------|----------------|
| KMS.1 | Customer managed policies should not allow KMS decrypt on all keys | MEDIUM | TerraformOpsBootstrap | Terraform bootstrap policy has broad KMS access. | Scope down to specific key ARNs at next IAM review. |
| IAM.21 | Customer managed policies should not allow wildcard actions | LOW | TerraformOpsBootstrap | Terraform bootstrap policy uses wildcards for provisioning flexibility. | Scope down. This policy was likely created during initial setup and never tightened. |

### S3 Logging

| Control | Title | Severity | Findings | Why Suppressed | Recommendation |
|---------|-------|----------|----------|----------------|----------------|
| S3.9 | S3 buckets should have access logging | MEDIUM | 6 buckets | Adds log storage costs and requires destination bucket. | **Enable for prod** — at minimum for terraform-state and frontend buckets. |

---

## Category 3: Logging & Monitoring Gaps

### CloudTrail Integration

| Control | Title | Severity | Why Suppressed | Recommendation |
|---------|-------|----------|----------------|----------------|
| CloudTrail.5 | CloudTrail should integrate with CloudWatch Logs | MEDIUM | Adds ~$0.50/GB ingestion cost. CloudTrail events are still in S3. | **Enable for prod.** Prerequisite for all CIS metric filter controls below. Enables real-time alerting on API activity. |
| CloudTrail.7 | CloudTrail S3 bucket should have access logging | LOW | Logging bucket for logging bucket. | Low priority. Enable if compliance requires it. |

### CloudWatch Log Metric Filters (CIS Benchmark)

All require CloudTrail.5 as a prerequisite. These create metric filters + SNS alarms for specific API activity.
**Enabling CloudTrail.5 first, then these 14 controls, would resolve 15 controls at once.**

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

**Estimated cost to enable all:** ~$2-5/mo (CloudTrail→CloudWatch ingestion + $0.10/alarm/mo x14)

---

## Summary

| Category | Controls | Highest Severity | Status |
|----------|----------|-----------------|--------|
| Already disabled (prior review) | 11 | HIGH | Disabled |
| Architecturally irrelevant (DISABLED) | 8 | HIGH | Disabled 2026-03-17 |
| Accepted risk (actionable) | 11 | CRITICAL (IAM.6) | Suppressed |
| Logging & monitoring gaps | 16 | MEDIUM | Suppressed |
| **Total** | **46** | | |

### Priority Remediation for Prod

1. **IAM.6 — Hardware MFA for root** (CRITICAL) — ~$25, one-time. Highest impact.
2. **RDS.5 — Enable multi-AZ** (MEDIUM) — ~$50/mo. Protects against DB downtime.
3. **RDS.6 — Enable enhanced monitoring** (LOW) — ~$3/mo. Better DB observability.
4. **CloudTrail.5 — CloudTrail→CloudWatch** (MEDIUM) — ~$2-5/mo. Unlocks 14 CIS controls.
5. **S3.9 — S3 access logging** (MEDIUM) — Free (just storage costs). Enable for key buckets.
6. **Network isolation** (EC2.9, RDS.46) — Significant architecture work. Move to private subnets.
7. **IAM tightening** (KMS.1, IAM.21) — Scope down TerraformOpsBootstrap policy.

### Comparison to Dev

| | Dev | Prod |
|---|---|---|
| Total failed controls | 48 | 35 |
| Already disabled | 14 (just now) | 11 (prior) |
| Critical findings | 0 | 1 (IAM.6 — root hardware MFA) |
| Secrets rotation | Failing | Disabled (accepted) |
| ECR tag immutability | Failing (10 repos) | Not failing |
| RDS logging | Failing (RDS.9, .36) | Not failing |
| Multi-AZ RDS | Failing | Failing |
