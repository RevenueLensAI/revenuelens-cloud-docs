# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the RevenueLens DevOps infrastructure repository containing two main components:
- **Terraform** (`revenuelens-cloud-terraform/`): AWS infrastructure provisioning
- **Ansible** (`revenuelens-cloud-ansible/`): Configuration management and deployment

The infrastructure supports a multi-environment SaaS application with dev and prod environments.

## Repository Structure

**IMPORTANT FOR CLAUDE:** This directory contains TWO separate Git repositories:
- `revenuelens-cloud-terraform/` has its own `.git` directory
- `revenuelens-cloud-ansible/` has its own `.git` directory
- There is NO git repository at the root `revenuelens/` level
- When working with files, ALWAYS navigate to the appropriate subdirectory before running git commands

```
revenuelens/
├── revenuelens-cloud-terraform/  # Infrastructure as Code (separate git repo)
│   ├── .git/                     # Git repository for Terraform code
│   ├── envs/                     # Environment-specific configurations
│   │   ├── dev/                  # Development environment
│   │   └── prod/                 # Production environment
│   └── modules/                  # Reusable Terraform modules
└── revenuelens-cloud-ansible/    # Configuration management (separate git repo)
    ├── .git/                     # Git repository for Ansible code
    ├── inventory/                # Environment inventories
    ├── playbooks/                # Ansible playbooks
    └── roles/                    # Ansible roles
```

## Common Commands

### Terraform Operations
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
```

### Ansible Operations
```bash
# Navigate to Ansible directory
cd revenuelens-cloud-ansible

# Infrastructure deployment (server setup, packages, configuration)
ansible-playbook -i inventory/prod/hosts.yml playbooks/site.yml

# Application infrastructure setup only (directories, certificates, logs)
ansible-playbook -i inventory/prod/hosts.yml playbooks/application-setup.yml

# Setup SSH keys for deployment
aws secretsmanager get-secret-value --secret-id revenuelens/deployer_pem --query SecretString --output text > ~/.ssh/revenuelens-deployer.pem
chmod 600 ~/.ssh/revenuelens-deployer.pem
```

### Developer Git Operations (Manual)
```bash
# SSH to development server
ssh -i ~/.ssh/revenuelens-dev-deployer.pem ubuntu@<DEV_IP>

# Clone repositories (one-time setup)
cd /app
git clone https://github.com/RevenueLensAI/revenuelens.ai.git
git clone https://github.com/RevenueLensAI/dashboard-react-app.git
git clone https://github.com/RevenueLensAI/dashboard-pmf.git

# Daily operations
cd /app/revenuelens.ai
git checkout feature-branch    # Switch branches
git pull origin feature-branch # Update code
```

### Git Operations
```bash
# For Terraform changes - navigate to terraform directory first
cd revenuelens-cloud-terraform
git status
git add .
git commit -m "Your terraform changes"

# For Ansible changes - navigate to ansible directory first  
cd revenuelens-cloud-ansible
git status
git add .
git commit -m "Your ansible changes"

# NEVER run git commands from the root revenuelens/ directory
# ALWAYS navigate to the specific component directory first
```

### AWS Operations
```bash
# Connect to EC2 instance via SSM (no SSH key needed)
aws ssm start-session --target <INSTANCE_ID>

# Retrieve database credentials
aws secretsmanager get-secret-value --secret-id revenuelens/db_master  # prod
aws secretsmanager get-secret-value --secret-id revenuelens-dev/db_master  # dev

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
# - CloudWatch alarm status
# - DNS resolution and SSL certificate validity
```

## Architecture

### Infrastructure Components (Terraform)
- **VPC**: Multi-AZ with public subnets and internet gateway
- **EC2**: Ubuntu 22.04 instances with Docker (t3.medium)
- **RDS**: PostgreSQL databases with Secrets Manager integration
- **ALB**: Application Load Balancer with HTTPS termination
- **S3**: Static frontend hosting with CloudFront CDN
- **Route 53**: DNS management with ACM certificates
- **Security**: IAM roles, Security Hub, GuardDuty, Config

### Configuration Management (Ansible)
- **Base Role**: System packages, user management, directory setup
- **Docker Role**: Docker and Docker Compose installation
- **Application Role**: Directory structure, certificates, log setup (infrastructure only)
- **Monitoring Role**: Health checks, log rotation, monitoring tools

### Application Deployment (Manual)
- **Git Operations**: Developers handle repository cloning, branch switching, updates
- **Code Deployment**: Manual control allows flexible branch testing and deployment timing
- **No Authentication Complexity**: Developers manage their own GitHub credentials

### Environment Configuration
**Development:**
- Domain: `dev.dashboard.revenuelens.ai`
- Project: `revenuelens-dev`
- Database secret: `revenuelens-dev/db_master`
- SSH key: `revenuelens-dev-deployer.pem`

**Production:**
- Domain: `dashboard.revenuelens.ai`
- Project: `revenuelens`
- Database secret: `revenuelens/db_master`
- SSH key: `revenuelens-deployer.pem`

## Integration Patterns

### Terraform → Ansible Workflow
1. **Terraform** provisions EC2 instances, RDS, networking, and security
2. **Ansible** configures the instances and deploys the application
3. **Secrets Manager** stores database credentials and SSH keys
4. **Route 53** provides DNS with environment-specific subdomains

### Security Model
- SSH access restricted to specific IP addresses in `terraform.tfvars`
- All secrets stored in AWS Secrets Manager
- SSL/TLS certificates managed by ACM
- IAM roles follow least-privilege principles
- Option for SSM Session Manager (no SSH keys required)

## Key Configuration Files

### Terraform
- `envs/{env}/terraform.tfvars`: Environment-specific variables
- `envs/{env}/backend.tf`: Remote state configuration
- `modules/`: Reusable infrastructure components

### Ansible
- `inventory/{env}/group_vars/all.yml`: Environment variables
- `inventory/{env}/hosts.yml`: Target server inventory
- `ansible.cfg`: Ansible configuration and SSH settings

## Development Workflow

1. **Infrastructure Changes**: Modify Terraform modules, plan and apply
2. **Configuration Changes**: Update Ansible roles and playbooks
3. **Application Deployment**: Run Ansible deployment playbooks
4. **State Management**: Terraform state stored in S3 with DynamoDB locking

## Access Patterns

### EC2 Access
1. **SSM Session Manager** (recommended): No SSH keys needed
2. **SSH with PEM key**: Requires key from Secrets Manager and IP allowlist
3. **AWS Console**: EC2 Connect with Session Manager

### Database Access
- Credentials stored in environment-specific Secrets Manager secrets
- RDS endpoints available via Terraform outputs
- PostgreSQL client access from EC2 instances

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
- Application code deployed from GitHub repository
- Log files stored in `/app/logs/` with rotation
- Health checks monitor application availability

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

- **[Registration Service](registration-service/README.md)**: OAuth 2.0 flow for HubSpot integration (port 8080)
- **[Webhook Service](webhook-service/README.md)**: Receives and processes external webhook events from HubSpot and Inboxly (port 8081)
- **[HubSync Service](hubsync-service/README.md)**: Synchronizes data between HubSpot and platform database
- **[CSV Generation Service](csv-generation-service/README.md)**: Generates CSV and JSON files from database used by the frontend to present data to customer
- **[Admin Service](admin-service/README.md)**: Internal admin portal for managing accounts and settings (port 9999)

## Important Conventions

- All services use the shared [`logger`](packages/logger/README.md ) package for consistent structured logging
- Database operations utilize the [`db-client`](packages/db-client/README.md ) package following repository pattern
- Environment configurations are centralized in [`config/`](config/README.md ) directory
- Services communicate through PostgreSQL for data persistence
- Redis is used for session persistence and interprocess communication with AOF and RDB backup mechanisms
- All services follow modular architecture with controllers, routes, utils, and views separation

## Development Notes

- The platform uses a microservices approach with service-specific Docker containers
- Database schemas are managed through the db-client package
- Redis persistence is configured for session survival across restarts (see REDIS_PERSISTENCE.md)
- Admin portal provides dashboard URL management with environment-aware generation
- HubSpot tokens are stored in PostgreSQL and shared across services for API access
- Slack integration available for critical log notifications with @channel mentions

## External Integrations

- **HubSpot**: OAuth flow, data synchronization, deal probability management
- **Inboxly**: Workspace mapping, campaign threshold configuration
- **PostgreSQL**: Primary data store for accounts, deals, probabilities, and tokens
- **Redis**: Session storage and caching layer, interprocess communication