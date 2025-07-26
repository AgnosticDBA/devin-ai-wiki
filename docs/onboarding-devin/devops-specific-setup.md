# DevOps-Specific Setup for Devin

This guide provides step-by-step instructions for configuring Devin to work effectively with your DevOps infrastructure, focusing on the specific tools and workflows used in this repository.

## 🎯 Prerequisites and Environment Setup

### Required Tools and Versions
Ensure your development environment has the following tools installed:

```bash
# Infrastructure as Code
terraform --version  # >= 1.0.0
ansible --version    # >= 2.9.0

# Cloud CLI Tools
openstack --version  # OpenStack CLI
vagrant --version    # >= 2.2.0

# Development Tools
git --version        # >= 2.20.0
python3 --version    # >= 3.8.0
pip3 --version       # Latest

# Optional but Recommended
docker --version     # >= 20.0.0
kubectl --version    # >= 1.20.0
```

### Environment Variables
Set up the following environment variables for OpenStack integration:

```bash
# OpenStack Authentication
export OS_AUTH_URL="https://your-openstack-endpoint:5000/v3"
export OS_PROJECT_NAME="your-project-name"
export OS_USERNAME="your-username"
export OS_PASSWORD="your-password"
export OS_REGION_NAME="RegionOne"
export OS_INTERFACE="public"
export OS_IDENTITY_API_VERSION="3"

# Terraform Variables
export TF_VAR_auth_url="$OS_AUTH_URL"
export TF_VAR_tenant_name="$OS_PROJECT_NAME"
export TF_VAR_user_name="$OS_USERNAME"
export TF_VAR_password="$OS_PASSWORD"
export TF_VAR_region="$OS_REGION_NAME"

# Ansible Configuration
export ANSIBLE_HOST_KEY_CHECKING="False"
export ANSIBLE_INVENTORY="inventory/hosts"
```

## 🔧 Repository-Specific Configuration

### Initial Setup Commands
```bash
# Clone and setup the repository
git clone https://github.com/AgnosticDBA/devops101-terraform-ansible-openstack.git
cd devops101-terraform-ansible-openstack

# Initialize Terraform
terraform init

# Validate Terraform configuration
terraform validate

# Check Ansible syntax
ansible-playbook --syntax-check playbook.yml

# Install Ansible dependencies
ansible-galaxy install -r requirements.yml  # if requirements.yml exists
```

### SSH Key Configuration
```bash
# Generate SSH key pair for OpenStack instances
ssh-keygen -t rsa -b 4096 -f ~/.ssh/openstack_key -N ""

# Update terraform.tfvars with your SSH key path
echo 'ssh_key_public = "~/.ssh/openstack_key.pub"' >> terraform.tfvars

# Ensure proper permissions
chmod 600 ~/.ssh/openstack_key
chmod 644 ~/.ssh/openstack_key.pub
```

### Network Configuration
```bash
# Verify OpenStack network connectivity
openstack network list
openstack flavor list
openstack image list

# Update terraform.tfvars with your network name
echo 'unique_network_name = "your-network-name"' >> terraform.tfvars
```

## 🚀 Devin Integration Setup

### GitHub Integration
1. **Repository Access**: Ensure Devin has access to your GitHub repository
2. **Branch Protection**: Configure appropriate branch protection rules
3. **Secrets Management**: Set up repository secrets for CI/CD

### Repository Secrets Configuration
```yaml
# Required GitHub Secrets
OPENSTACK_AUTH_URL: "https://your-openstack-endpoint:5000/v3"
OPENSTACK_USERNAME: "your-username"
OPENSTACK_PASSWORD: "your-password"
OPENSTACK_PROJECT_NAME: "your-project"
OPENSTACK_REGION: "RegionOne"

# SSH Keys
ANSIBLE_SSH_PRIVATE_KEY: "-----BEGIN OPENSSH PRIVATE KEY-----..."

# Optional: Notification Integration
SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/..."
```

### Devin Workspace Configuration
Create a `.devin/` directory with configuration files:

```bash
mkdir -p .devin
```

#### .devin/config.yml
```yaml
project:
  name: "devops101-terraform-ansible-openstack"
  type: "infrastructure"
  description: "OpenStack infrastructure automation with Terraform and Ansible"

tools:
  primary:
    - terraform
    - ansible
    - openstack
  secondary:
    - docker
    - vagrant
    - git

environments:
  development:
    terraform_workspace: "dev"
    ansible_inventory: "inventory/dev/hosts"
  staging:
    terraform_workspace: "staging"
    ansible_inventory: "inventory/staging/hosts"
  production:
    terraform_workspace: "prod"
    ansible_inventory: "inventory/prod/hosts"

conventions:
  terraform:
    naming: "snake_case"
    file_organization: "purpose-based"
  ansible:
    style: "yaml"
    role_structure: "galaxy"
  git:
    branch_naming: "feature/description"
    commit_format: "conventional"
```

#### .devin/workflows.yml
```yaml
workflows:
  infrastructure_change:
    steps:
      - "terraform plan"
      - "review plan output"
      - "terraform apply"
      - "ansible-playbook configuration"
      - "validate deployment"
      - "update documentation"

  new_server_deployment:
    steps:
      - "update terraform configuration"
      - "terraform plan and apply"
      - "update ansible inventory"
      - "run ansible playbook"
      - "verify server configuration"
      - "add to monitoring"

  security_update:
    steps:
      - "identify affected systems"
      - "test in development"
      - "apply to staging"
      - "validate security posture"
      - "deploy to production"
      - "verify compliance"
```

## 🔍 Testing and Validation Setup

### Local Testing Environment
```bash
# Set up Vagrant for local testing
vagrant up

# Test Ansible playbook locally
ansible-playbook -i vagrant_inventory playbook.yml

# Validate Terraform configuration
terraform plan -var-file="dev.tfvars"
```

### CI/CD Pipeline Setup
Create `.github/workflows/infrastructure-ci.yml`:

```yaml
name: Infrastructure CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: terraform fmt -check
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        run: terraform plan -var-file="dev.tfvars"
        env:
          TF_VAR_auth_url: ${{ secrets.OPENSTACK_AUTH_URL }}
          TF_VAR_user_name: ${{ secrets.OPENSTACK_USERNAME }}
          TF_VAR_password: ${{ secrets.OPENSTACK_PASSWORD }}

  ansible:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install Ansible
        run: |
          pip install ansible ansible-lint
      
      - name: Ansible Syntax Check
        run: ansible-playbook --syntax-check playbook.yml
      
      - name: Ansible Lint
        run: ansible-lint playbook.yml
```

## 🛡️ Security Configuration

### Ansible Vault Setup
```bash
# Create vault password file (keep secure)
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass

# Add to .gitignore
echo ".vault_pass" >> .gitignore

# Create encrypted variables
ansible-vault create group_vars/all/vault.yml --vault-password-file .vault_pass
```

### Security Scanning Integration
```bash
# Install security scanning tools
pip install checkov  # Terraform security scanning
pip install ansible-lint  # Ansible best practices

# Run security scans
checkov -f terraform-instances.tf
ansible-lint playbook.yml
```

## 📊 Monitoring and Observability Setup

### Prometheus Configuration
```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - 'web1:9100'
        - 'web2:9100'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager:9093'
```

### Grafana Dashboard Setup
```json
{
  "dashboard": {
    "title": "OpenStack Infrastructure",
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
          }
        ]
      }
    ]
  }
}
```

## 🎯 Devin-Specific Best Practices

### Communication Templates
When working with Devin on this repository, use these templates:

#### Infrastructure Change Request
```
Task: [Brief description]

Current Infrastructure:
- [List current resources from terraform state]
- [Reference existing files: terraform-instances.tf, playbook.yml]

Requirements:
- [Specific technical requirements]
- [Security considerations]
- [Performance requirements]

Implementation:
- [Terraform changes needed]
- [Ansible configuration updates]
- [Testing approach]

Validation:
- [How to verify the changes work]
- [Expected outcomes]
```

#### Troubleshooting Request
```
Issue: [Description of the problem]

Environment:
- [Which environment: dev/staging/prod]
- [Affected resources]
- [Error messages or symptoms]

Context:
- [Recent changes made]
- [Current state of infrastructure]
- [Relevant log files or outputs]

Expected Resolution:
- [What should be working]
- [Success criteria]
```

### File Organization Guidelines
```
# When asking Devin to modify files, reference:
terraform-instances.tf    # Server definitions
terraform-networks.tf     # Network configuration
terraform-provider.tf     # Provider setup
playbook.yml             # Main Ansible playbook
roles/                   # Ansible roles directory
inventory/               # Environment-specific inventories
group_vars/              # Ansible variables
```

This setup ensures Devin can effectively work with your DevOps infrastructure while following your organization's standards and security requirements.
