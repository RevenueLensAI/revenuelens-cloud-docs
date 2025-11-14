# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the RevenueLens DevOps infrastructure repository containing four main components:
- **Root Directory** (`revenuelens/`): Main git repository for project coordination and documentation
- **Infrastructure Terraform** (`revenuelens-cloud-terraform/`): AWS infrastructure provisioning (separate git repo)
- **Configuration Terraform** (`revenuelens-config-terraform/`): AWS-managed configuration via SSM Parameter Store and Secrets Manager (separate git repo)
- **Ansible** (`revenuelens-cloud-ansible/`): Server configuration management and deployment (separate git repo)

The infrastructure supports a multi-environment SaaS application with dev and prod environments.

## Repository Structure

**IMPORTANT FOR CLAUDE:** This directory contains FOUR separate Git repositories:
- Root `revenuelens/` has its own `.git` directory (this repo - documentation)
- `revenuelens-cloud-terraform/` has its own `.git` directory (infrastructure)
- `revenuelens-config-terraform/` has its own `.git` directory (application configuration)
- `revenuelens-cloud-ansible/` has its own `.git` directory (server configuration)
- When working with files, ALWAYS navigate to the appropriate subdirectory before running git commands

```
revenuelens/
├── .git/                         # Git repository for project documentation
├── CLAUDE.md                     # This file
├── revenuelens-cloud-terraform/  # Infrastructure as Code (separate git repo)
│   ├── .git/                     # Git repository for Terraform infrastructure
│   ├── envs/                     # Environment-specific configurations
│   │   ├── dev/                  # Development environment
│   │   └── prod/                 # Production environment
│   └── modules/                  # Reusable Terraform modules
│       ├── vpc/                  # VPC, subnets, security groups
│       ├── rds/                  # PostgreSQL database
│       ├── ec2_docker_web/       # EC2 instances with Docker
│       ├── monitoring/           # CloudWatch alarms (prod only)
│       ├── route53/              # DNS and SSL certificates
│       ├── s3/                   # Static asset storage
│       ├── cloudfront/           # CDN
│       └── secrets_manager/      # Credential storage
├── revenuelens-config-terraform/ # Application configuration (separate git repo)
│   ├── .git/                     # Git repository for Terraform configuration
│   ├── envs/                     # Environment-specific configurations
│   │   ├── dev/                  # Development configuration
│   │   │   ├── *.auto.tfvars     # Service-specific configuration files
│   │   │   ├── main.tf           # Module invocations
│   │   │   └── variables.tf     # Variable declarations
│   │   └── prod/                 # Production configuration
│   │       └── (same structure)
│   └── modules/                  # Reusable configuration modules
│       ├── config-store/         # SSM Parameter Store and Secrets Manager
│       └── iam-permissions/      # IAM policy for EC2 config access
└── revenuelens-cloud-ansible/    # Server configuration (separate git repo)
    ├── .git/                     # Git repository for Ansible code
    ├── inventory/                # Environment inventories with EBS volume configs
    │   ├── dev/group_vars/all.yml # Dev environment variables
    │   └── prod/group_vars/all.yml# Prod environment variables
    ├── playbooks/                # Ansible playbooks
    └── roles/                    # Ansible roles
        ├── base/                 # System packages, users
        ├── docker/               # Docker and Docker Compose
        ├── application/          # App directories, certs, logs
        ├── ebs_volumes/          # EBS volume mounting and formatting
        ├── config-sync/          # Sync config from AWS to /app/config/*.env
        └── monitoring/           # Log rotation, health checks
```

## Common Commands

### Infrastructure Terraform Operations (revenuelens-cloud-terraform)
```bash
# Navigate to environment directory
cd revenuelens-cloud-terraform/envs/prod  # or dev

# Initialize and plan
terraform init
terraform plan -var-file=terraform.tfvars

# Apply infrastructure changes
terraform apply -var-file=terraform.tfvars

# Retrieve outputs (instance IPs, URLs, etc.)
terraform output

# Show current state
terraform show

# Destroy infrastructure (use with caution)
terraform destroy -var-file=terraform.tfvars
```

### Configuration Terraform Operations (revenuelens-config-terraform)
```bash
# Navigate to environment directory
cd revenuelens-config-terraform/envs/prod  # or dev

# Initialize Terraform
terraform init

# Plan configuration changes (auto-loads *.auto.tfvars files)
terraform plan

# Apply configuration changes to AWS SSM Parameter Store
terraform apply

# View created SSM parameters
terraform output -json ssm_parameter_names

# View IAM policy ARN
terraform output iam_policy_arn

# IMPORTANT: After terraform apply, config syncs automatically to EC2 every 15 minutes
# Or trigger manually: ssh ubuntu@instance 'sudo /usr/local/bin/revenuelens-config-sync.sh dev'
```

### Ansible Operations
```bash
# Navigate to Ansible directory
cd revenuelens-cloud-ansible

# Full infrastructure deployment (server setup, packages, configuration)
ansible-playbook -i inventory/prod/hosts.yml playbooks/site.yml

# Application infrastructure setup only (directories, certificates, logs)
ansible-playbook -i inventory/prod/hosts.yml playbooks/application-setup.yml

# EBS volume setup (mounting and formatting)
ansible-playbook -i inventory/prod/hosts.yml playbooks/ebs-setup.yml

# Deploy config-sync mechanism (fetches config from AWS SSM/Secrets Manager)
ansible-playbook -i inventory/prod/hosts.yml playbooks/config-sync-setup.yml

# Setup SSH keys for deployment
aws secretsmanager get-secret-value --secret-id revenuelens/deployer_pem --query SecretString --output text > ~/.ssh/revenuelens-deployer.pem
chmod 600 ~/.ssh/revenuelens-deployer.pem
```

### Git Operations
```bash
# For root project documentation - navigate to root directory
cd /home/gwichman/revenuelens
git status
git add .
git commit -m "Update documentation"
git push

# For infrastructure Terraform changes - navigate to cloud-terraform directory
cd revenuelens-cloud-terraform
git status
git add .
git commit -m "Update infrastructure"
git push

# For configuration Terraform changes - navigate to config-terraform directory
cd revenuelens-config-terraform
git status
git add .
git commit -m "Update application configuration"
git push

# For Ansible changes - navigate to ansible directory
cd revenuelens-cloud-ansible
git status
git add .
git commit -m "Update server configuration"
git push

# NEVER run git commands from the wrong directory
# ALWAYS navigate to the specific component directory first
```

### AWS Operations
**IMPORTANT: Run these commands from the appropriate Terraform environment directory to use correct AWS profile:**
- Dev commands: `cd revenuelens-cloud-terraform/envs/dev/`
- Prod commands: `cd revenuelens-cloud-terraform/envs/prod/`

```bash
# Connect to EC2 instance via SSM (no SSH key needed)
aws ssm start-session --target <INSTANCE_ID> --profile <revenuelens-dev|revenuelens-prod>

# Retrieve database credentials
aws secretsmanager get-secret-value --secret-id revenuelens/db_master --profile revenuelens-prod  # prod
aws secretsmanager get-secret-value --secret-id revenuelens-dev/db_master --profile revenuelens-dev  # dev

# SSH to EC2 instance (requires SSH key and IP allowlist)
ssh -i ~/.ssh/revenuelens-deployer.pem ubuntu@<PUBLIC_IP>
```

### Health Check Scripts
```bash
# Production health check (run after maintenance or changes)
cd revenuelens-cloud-terraform/envs/prod
./prod-health-check.sh

# Development health check (run after maintenance or changes)
cd revenuelens-cloud-terraform/envs/dev
./dev-health-check.sh

# Health checks verify:
# - EC2 instance status and health checks
# - ALB target group health for all services
# - HTTP endpoint responses (registration, webhook, deals, pmf)
# - CloudWatch alarm status (prod only)
# - DNS resolution and SSL certificate validity
```

## Architecture

### Infrastructure Components (Terraform)
- **VPC**: Multi-AZ with public subnets and internet gateway
- **EC2**: Ubuntu 22.04 instances with Docker (t3.medium)
- **EBS Volumes**: Dedicated volumes for /var, /tmp, /home, /var/lib/docker, /app, and swap
- **RDS**: PostgreSQL databases (db.t4g.small for prod, db.t3.micro for dev) with Secrets Manager integration
- **ALB**: Application Load Balancer with HTTPS termination
- **S3**: Static frontend hosting with CloudFront CDN
- **Route 53**: DNS management with ACM certificates
- **Security**: IAM roles, Security Hub, GuardDuty, Config (configurable)
- **Monitoring**: Comprehensive CloudWatch monitoring (production only)
- **Email Security**: SES domain identity with SPF, DMARC, and DKIM records

### Application Configuration (Terraform - config-terraform)
- **SSM Parameter Store**: Non-sensitive configuration (database hosts, ports, log levels, feature flags) - FREE
- **Secrets Manager**: Sensitive credentials (API keys, passwords, tokens) - ~$0.40/secret/month
- **IAM Policies**: EC2 instance permissions to read configuration
- **Config Files**: 12 service-specific .auto.tfvars files (postgres, redis, logger, hubspot, etc.)
- **Automatic Sync**: Config synced to EC2 every 15 minutes to `/app/config/*.env`

### Server Configuration (Ansible - cloud-ansible)
- **Base Role**: System packages, user management, directory setup
- **Docker Role**: Docker and Docker Compose installation (version 2.20.2)
- **Application Role**: Directory structure (/app/revenuelens.ai, /app/dashboard-react-app, /app/dashboard-pmf), certificates, log setup
- **EBS Volumes Role**: Mounting and formatting dedicated EBS volumes
- **Config-Sync Role**: Deploys sync script to fetch config from AWS every 15 minutes
- **Monitoring Role**: Health checks, log rotation, monitoring tools

### Application Deployment (Manual)
- **Git Operations**: Developers handle repository cloning, branch switching, updates
- **Code Deployment**: Manual control allows flexible branch testing and deployment timing
- **No Authentication Complexity**: Developers manage their own GitHub credentials
- **Service Ports**: 8080 (main), 8081 (webhook), 8000 (API), 8001 (additional microservice)

### Environment Configuration
**Development:**
- Domain: `dev.dashboard.revenuelens.ai`
- Project: `revenuelens-dev`
- Database: `revenuelens-dev-pg.cu9t5w8k8yue.us-west-2.rds.amazonaws.com`
- Database secret: `revenuelens-dev/db_master`
- SSH key: `revenuelens-dev-deployer.pem`
- Instance Type: t3.medium
- VPC CIDR: 10.127.0.0/16

**Production:**
- Domain: `dashboard.revenuelens.ai`
- Project: `revenuelens`
- Database: `revenuelens-pg.cu9t5w8k8yue.us-west-2.rds.amazonaws.com`
- Database secret: `revenuelens/db_master`
- SSH key: `revenuelens-deployer.pem`
- Instance Type: t3.medium
- VPC CIDR: 10.128.0.0/16

## Integration Patterns

### Infrastructure Deployment Workflow
1. **Infrastructure Terraform (cloud-terraform)** provisions EC2 instances, RDS, networking, security, and EBS volumes
2. **Configuration Terraform (config-terraform)** creates SSM parameters, IAM policies for config access
3. **Ansible (cloud-ansible)** configures the instances, mounts EBS volumes, deploys config-sync script
4. **Config-Sync** runs every 15 minutes on EC2, fetching configuration from SSM/Secrets Manager to `/app/config/*.env`
5. **Application** reads configuration from .env files in /app/config/

### Configuration Management Pattern
**Traditional (old)**: Environment variables stored in Ansible inventory, manually updated on EC2 via Ansible playbooks
**New AWS-Managed**:
- Engineers edit `.auto.tfvars` files in config-terraform repo
- `terraform apply` pushes to AWS SSM Parameter Store (free) or Secrets Manager (for secrets)
- EC2 instances automatically sync config every 15 minutes via cron
- No Ansible runs needed for config changes
- Config changes tracked in Git with proper review process

### Security Model
- SSH access restricted to specific IP addresses in `terraform.tfvars` (ssh_cidrs and admin_cidrs)
- All secrets stored in AWS Secrets Manager
- SSL/TLS certificates managed by ACM
- IAM roles follow least-privilege principles
- Option for SSM Session Manager (no SSH keys required)
- Email security with SPF, DMARC, and DKIM records

## Production Monitoring Setup

### Comprehensive Monitoring Module (Production Only)
The monitoring module provides comprehensive infrastructure and application monitoring for the production environment.

**RDS PostgreSQL Database Monitoring:**
- **Connection Failures**: > 5 failures in 5 minutes
- **CPU Utilization**: High alert (> 80% for 15 minutes) and Critical alert (> 90% for 5 minutes)
- **Database Connections**: > 16 connections (80% of max connections for db.t4g.small)
- **Storage Space**: < 10 GB free space remaining
- **Freeable Memory**: < 100 MB available for 5 minutes
- **Read/Write Latency**: > 200ms average over 5 minutes

**Application Load Balancer Monitoring:**
- **Unhealthy Targets**: Any unhealthy target for 1 minute (immediate alert)
- **Response Time**: > 3 seconds average over 5 minutes
- **Rejected Connections**: Any rejected connections (configuration issues)
- **5xx Errors**: Configurable threshold (default 3 requests in 5 minutes)
- **4xx Errors**: Configurable threshold (default 15 requests in 5 minutes) - Warning level

**SSL Certificate Management:**
- **Certificate Expiration**: Alert when expiring within 30 days

**EBS Filesystem Monitoring:**
- **IOPS Utilization**: > 80% of EBS volume IOPS capacity
- **Throughput Utilization**: Monitors bandwidth usage approaching EBS limits
- **Average Latency**: Tracks I/O response time
- **Queue Depth**: Monitors pending I/O operations indicating saturation

**Composite Alarms:**
- **"App Down"**: Combines all unhealthy target alerts to avoid duplicate notifications
- **"RDS Critical"**: Combines critical database issues (CPU, storage, memory, connections)

**Notification Channels:**
- Email notifications to `devops-admin@revenuelens.ai`
- Slack integration (channel: C099E5PBDGV, workspace: T08SSJ4LYHG)

**Cost:** ~$3-5/month for comprehensive CloudWatch monitoring

### Monitoring Commands
```bash
# View all CloudWatch alarms status (run from envs/prod/)
aws cloudwatch describe-alarms --alarm-name-prefix "revenuelens-" --profile revenuelens-prod

# View specific alarm categories
aws cloudwatch describe-alarms --alarm-names revenuelens-rds-cpu-critical revenuelens-rds-storage-low --profile revenuelens-prod

# Check composite alarms (avoid duplicate alerts)
aws cloudwatch describe-alarms --alarm-names revenuelens-app-down-composite revenuelens-rds-critical-composite --profile revenuelens-prod

# Test SNS notifications (sends test message)
aws sns publish --topic-arn $(terraform output -raw monitoring_sns_topic) --message "Test monitoring alert" --profile revenuelens-prod

# View ALB target group health
aws elbv2 describe-target-health --target-group-arn $(aws elbv2 describe-target-groups --names revenuelens-tg-registration --query 'TargetGroups[0].TargetGroupArn' --output text) --profile revenuelens-prod
```

## Email Security Configuration

### Overview
The production environment includes comprehensive email security setup with AWS SES integration for future application email sending (password resets, notifications, etc.).

**Important Note**: CloudWatch alerts use SNS and work independently of SES. This email configuration is for application-sent emails only.

### Components Configured

**SPF Record (`dashboard.revenuelens.ai` TXT):**
- `v=spf1 include:amazonses.com -all`
- Authorizes only Amazon SES to send emails on behalf of the domain
- Uses `-all` (hard fail) policy for maximum security

**DMARC Record (`_dmarc.dashboard.revenuelens.ai` TXT):**
- `v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@revenuelens.ai; ruf=mailto:dmarc-failures@revenuelens.ai; fo=1`
- Sets policy to quarantine failed messages
- Sends aggregate reports to `dmarc-reports@revenuelens.ai`
- Sends forensic reports to `dmarc-failures@revenuelens.ai`

**DKIM Records (3x CNAME):**
- Automatically generated by SES domain verification
- Format: `{token}._domainkey.dashboard.revenuelens.ai` → `{token}.dkim.amazonses.com`
- Provides cryptographic email authentication

**SES Domain Identity:**
- Domain: `dashboard.revenuelens.ai`
- Automatically verified via Route 53 DNS records
- Ready for application integration when needed

### SES Commands
```bash
# Check SES domain verification status (run from envs/prod/)
aws ses get-identity-verification-attributes --identities dashboard.revenuelens.ai --profile revenuelens-prod

# View SES domain DKIM attributes
aws ses get-identity-dkim-attributes --identities dashboard.revenuelens.ai --profile revenuelens-prod

# List all verified SES identities
aws ses list-identities --identity-type Domain --profile revenuelens-prod
```

## Key Configuration Files

### Infrastructure Terraform (cloud-terraform)
- `envs/{env}/terraform.tfvars`: Environment-specific variables including ssh_cidrs, admin_cidrs, instance_type, custom_domain, aws_profile
- `envs/{env}/backend.tf`: Remote state configuration (S3 + DynamoDB)
- `modules/`: Reusable infrastructure components (VPC, RDS, EC2, ALB, etc.)

### Configuration Terraform (config-terraform)
- `envs/{env}/*.auto.tfvars`: Service-specific configuration files (postgres, redis, logger, hubspot, etc.)
- `envs/{env}/main.tf`: Merges all service variables and invokes modules
- `envs/{env}/variables.tf`: Variable declarations for all 12 services
- `envs/{env}/backend.tf`: Remote state configuration (S3 + DynamoDB)
- `modules/config-store/`: Creates SSM parameters and Secrets Manager entries
- `modules/iam-permissions/`: Creates IAM policy for EC2 config access

### Ansible (cloud-ansible)
- `inventory/{env}/group_vars/all.yml`: Environment variables including database endpoints, EBS volume IDs, service ports
- `inventory/{env}/hosts.yml`: Target server inventory
- `ansible.cfg`: Ansible configuration and SSH settings
- `roles/config-sync/defaults/main.yml`: Config sync settings (interval: 15 minutes, config dir: /app/config)

## EBS Volume Configuration

### Development Environment
Dedicated EBS volumes for filesystem separation and performance:
- **vol-0bbd6b43a0abdeb69**: `/var` (8GB ext4)
- **vol-0d67bd294c0aced45**: `/tmp` (4GB ext4)
- **vol-0cfa2ced53cff7e24**: `/home` (5GB ext4)
- **vol-06e718cff41856932**: `/var/lib/docker` (15GB ext4)
- **vol-073549949e7132597**: `/app` (10GB ext4)
- **vol-083e962ab43429af4**: Swap volume (4GB)

### Production Environment
Similar EBS volume structure with production-specific volume IDs (configured in Ansible inventory).

## Code Preferences

### Ansible Best Practices
- **ALWAYS prefer Ansible modules over shell/command tasks**
- Use `ansible.builtin.git` instead of shell git commands
- Use `ansible.builtin.file` instead of shell mkdir/chmod commands
- Use `ansible.builtin.copy` instead of shell cp commands
- Use `ansible.builtin.template` instead of shell sed/awk operations
- Only use shell/command when no appropriate Ansible module exists
- Ansible modules provide better idempotency, error handling, and logging

## Troubleshooting Guidelines

### Documentation Source Priority
When troubleshooting problems, prioritize information sources in this order:
1. **Official documentation** (docs.ansible.com, docs.aws.amazon.com, terraform.io)
2. **Official project repositories** (GitHub repos from ansible, hashicorp, aws)
3. **Vendor-specific guides** (AWS whitepapers, Ansible best practices)
4. **Community sources** (Stack Overflow, forums) - use with caution
5. **Third-party tutorials** - verify against official sources

### Research Best Practices
- Always cross-reference solutions with official documentation
- Verify compatibility with current versions of tools (Ansible, Terraform, AWS CLI)
- Test solutions in development environment before applying to production
- Document the root cause and official solution for future reference

## Notes

- Both environments use identical EC2 + ALB architecture
- SSL certificates are environment-specific (ACM managed)
- Application code deployed from GitHub repository manually by developers
- Log files stored in `/app/logs/` with rotation
- Health checks monitor application availability
- Production includes comprehensive monitoring and email security
- Development uses manual monitoring for cost savings

# RevenueLens.ai Backend Application

The following describes the application deployed on this AWS infrastructure. It is a microservices platform for revenue intelligence and business analytics with integrated HubSpot synchronization and Inboxly campaign management.

## Architecture

- **Backend**: Node.js with Express across multiple microservices
- **Database**: PostgreSQL with Redis for session storage and caching
- **Authentication**: HubSpot OAuth 2.0 integration (registration-service)
- **Containerization**: Docker with Docker Compose orchestration
- **Logging**: Centralized structured logging with Slack notifications
- **External Integrations**: HubSpot API and Webhooks, Inboxly Webhooks

## Key Services

- **Registration Service**: OAuth 2.0 flow for HubSpot integration (port 8080)
- **Webhook Service**: Receives and processes external webhook events from HubSpot and Inboxly (port 8081)
- **HubSync Service**: Synchronizes data between HubSpot and platform database
- **CSV Generation Service**: Generates CSV and JSON files from database used by the frontend to present data to customer
- **Admin Service**: Internal admin portal for managing accounts and settings (port 9999)

## Important Conventions

- All services use shared logging packages for consistent structured logging
- Database operations utilize shared db-client packages following repository pattern
- Environment configurations are centralized in config directories
- Services communicate through PostgreSQL for data persistence
- Redis is used for session persistence and interprocess communication with AOF and RDB backup mechanisms
- All services follow modular architecture with controllers, routes, utils, and views separation

## Development Notes

- The platform uses a microservices approach with service-specific Docker containers
- Database schemas are managed through the db-client package
- Redis persistence is configured for session survival across restarts
- Admin portal provides dashboard URL management with environment-aware generation
- HubSpot tokens are stored in PostgreSQL and shared across services for API access
- Slack integration available for critical log notifications with @channel mentions

## External Integrations

- **HubSpot**: OAuth flow, data synchronization, deal probability management
- **Inboxly**: Workspace mapping, campaign threshold configuration
- **PostgreSQL**: Primary data store for accounts, deals, probabilities, and tokens
- **Redis**: Session storage and caching layer, interprocess communication

Ansible commands should be run by executing the appropriate GitHub Action.