# Repository Setup for Devin

Setting up your repository properly ensures Devin can understand your infrastructure code, follow your conventions, and make meaningful contributions to your DevOps workflows.

## 🎯 Initial Repository Configuration

### Repository Structure Best Practices

For optimal Devin integration, organize your repository with clear, logical structure:

```
devops101-terraform-ansible-openstack/
├── README.md                    # Project overview and quick start
├── docs/                        # Documentation (including this wiki)
├── terraform/                   # Terraform configurations
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
├── ansible/                     # Ansible configurations
│   ├── playbook.yml
│   ├── inventory/
│   ├── roles/
│   └── group_vars/
├── scripts/                     # Automation scripts
├── .github/                     # GitHub workflows
│   └── workflows/
└── environments/                # Environment-specific configs
    ├── dev/
    ├── staging/
    └── prod/
```

### Essential Documentation Files

#### README.md Template
```markdown
# Project Name

## Overview
Brief description of your infrastructure and its purpose.

## Prerequisites
- Terraform >= 1.0
- Ansible >= 2.9
- OpenStack CLI tools
- Required credentials and access

## Quick Start
1. Clone repository
2. Configure credentials
3. Run terraform init
4. Execute deployment

## Architecture
High-level infrastructure diagram and component description.

## Devin AI Integration
This repository is optimized for Devin AI assistance. See [docs/](./docs/) for:
- Setup guidelines
- Best practices
- Common workflows
```

#### CONTRIBUTING.md
```markdown
# Contributing Guidelines

## Development Workflow
1. Create feature branch
2. Make changes following our conventions
3. Test in development environment
4. Submit pull request

## Code Standards
- Terraform: Follow HashiCorp style guide
- Ansible: Use YAML best practices
- Documentation: Update relevant docs

## Working with Devin AI
- Provide clear context in requests
- Reference existing patterns
- Include validation steps
- Follow security guidelines
```

## 🔧 Configuration Files for Devin

### .devinconfig (Optional)
Create a configuration file to help Devin understand your project:

```yaml
# .devinconfig
project:
  type: "infrastructure"
  primary_tools:
    - terraform
    - ansible
    - openstack
  
conventions:
  terraform:
    naming: "snake_case"
    file_structure: "modular"
  ansible:
    style: "yaml"
    role_structure: "galaxy"

environments:
  - development
  - staging
  - production

documentation:
  location: "docs/"
  style: "markdown"
  
security:
  secrets_management: "environment_variables"
  sensitive_files:
    - "*.tfvars"
    - "inventory/*/group_vars/*/vault.yml"
```

### Environment Configuration

#### terraform.tfvars.example
```hcl
# OpenStack Configuration
auth_url = "https://your-openstack-endpoint:5000/v3"
tenant_name = "your-tenant"
user_name = "your-username"
password = "your-password"
region = "RegionOne"

# Instance Configuration
ssh_key_public = "~/.ssh/id_rsa.pub"
unique_network_name = "your-network-name"

# Naming Convention
environment = "dev"
project_name = "devops101"
```

#### ansible.cfg
```ini
[defaults]
host_key_checking = False
inventory = inventory/hosts
roles_path = roles
retry_files_enabled = False
gathering = smart
fact_caching = memory

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

## 📋 Setting Up Devin Access

### GitHub Integration
1. **Repository Access**: Ensure Devin has access to your repository through GitHub integration
2. **Branch Protection**: Configure branch protection rules that work with Devin's workflow
3. **Secrets Management**: Set up repository secrets for CI/CD integration

### Required Secrets
```yaml
# GitHub Repository Secrets
OPENSTACK_AUTH_URL: "https://your-openstack-endpoint:5000/v3"
OPENSTACK_USERNAME: "your-username"
OPENSTACK_PASSWORD: "your-password"
OPENSTACK_TENANT_NAME: "your-tenant"
OPENSTACK_REGION: "RegionOne"

# SSH Keys for Ansible
ANSIBLE_SSH_PRIVATE_KEY: "-----BEGIN OPENSSH PRIVATE KEY-----..."

# Notification Webhooks
SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/..."
```

## 🛡️ Security Configuration

### Sensitive File Management
Create `.gitignore` to protect sensitive information:

```gitignore
# Terraform
*.tfstate
*.tfstate.*
*.tfvars
.terraform/
.terraform.lock.hcl

# Ansible
inventory/*/group_vars/*/vault.yml
*.retry

# SSH Keys
*.pem
*.key
id_rsa*

# Environment Files
.env
.env.local

# IDE Files
.vscode/
.idea/

# OS Files
.DS_Store
Thumbs.db
```

### Vault Configuration for Ansible
```yaml
# group_vars/all/vault.yml (encrypted)
vault_openstack_password: "encrypted_password"
vault_database_password: "encrypted_db_password"
vault_api_keys:
  monitoring: "encrypted_monitoring_key"
  backup: "encrypted_backup_key"
```

## 🔄 Workflow Integration

### Pre-commit Hooks
Set up pre-commit hooks to maintain code quality:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.81.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs

  - repo: https://github.com/ansible/ansible-lint
    rev: v6.17.2
    hooks:
      - id: ansible-lint
```

### GitHub Actions Workflow
```yaml
# .github/workflows/infrastructure-ci.yml
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
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: terraform fmt -check
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        run: terraform plan
        env:
          TF_VAR_auth_url: ${{ secrets.OPENSTACK_AUTH_URL }}
          TF_VAR_user_name: ${{ secrets.OPENSTACK_USERNAME }}
          TF_VAR_password: ${{ secrets.OPENSTACK_PASSWORD }}

  ansible:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install Ansible
        run: pip install ansible ansible-lint
      
      - name: Ansible Syntax Check
        run: ansible-playbook --syntax-check playbook.yml
      
      - name: Ansible Lint
        run: ansible-lint playbook.yml
```

## 📊 Monitoring and Observability Setup

### Infrastructure Monitoring
```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - 'web1:9100'
        - 'web2:9100'
        - 'monitoring:9100'

  - job_name: 'openstack-exporter'
    static_configs:
      - targets:
        - 'monitoring:9180'
```

### Logging Configuration
```yaml
# logging/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/apache2/*.log
    - /var/log/syslog
    - /var/log/auth.log

output.elasticsearch:
  hosts: ["elasticsearch:9200"]

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
```

## 🎯 Best Practices for Devin Integration

### Code Organization
1. **Modular Structure**: Organize code in logical modules
2. **Clear Naming**: Use descriptive names for resources and variables
3. **Documentation**: Comment complex configurations
4. **Version Control**: Tag releases and maintain changelog

### Devin-Friendly Patterns
1. **Consistent Formatting**: Use automated formatting tools
2. **Clear Dependencies**: Document resource relationships
3. **Error Handling**: Include proper error handling and rollback procedures
4. **Testing**: Implement infrastructure testing where possible

### Communication Guidelines
1. **Context**: Always provide context about existing infrastructure
2. **Requirements**: Be specific about what needs to be accomplished
3. **Constraints**: Mention any limitations or requirements
4. **Validation**: Include steps to verify changes work correctly

This setup ensures Devin can effectively understand your infrastructure, follow your conventions, and contribute meaningfully to your DevOps workflows.
