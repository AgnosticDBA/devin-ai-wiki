# Knowledge Onboarding for Devin

Teaching Devin about your infrastructure, conventions, and organizational practices is crucial for effective collaboration on DevOps tasks. This guide helps you systematically onboard Devin to your specific environment.

## 🎯 Infrastructure Knowledge Transfer

### Current Infrastructure Overview
Start by providing Devin with a comprehensive understanding of your existing infrastructure:

```
Our OpenStack Infrastructure Overview:

Current Resources:
- 2 web servers (web1, web2) running Ubuntu 18.04
- 1 security group allowing HTTP/HTTPS traffic
- 1 private network with CIDR 10.0.1.0/24
- SSH key pair for server access
- Load balancer (planned)

Architecture Pattern:
- Web tier: Apache2 servers behind load balancer
- Database tier: MySQL (to be implemented)
- Monitoring: Prometheus + Grafana (planned)

Network Design:
- Public network: External connectivity
- Private network: Internal communication
- Security groups: Service-specific access rules

File Structure:
- terraform-instances.tf: Server definitions
- terraform-networks.tf: Network configuration
- terraform-provider.tf: OpenStack provider setup
- playbook.yml: Main Ansible configuration
- roles/: Modular Ansible roles
```

### Technology Stack Documentation
```yaml
# Technology Stack
Infrastructure:
  cloud_provider: "OpenStack"
  regions: ["RegionOne"]
  availability_zones: ["nova"]

Automation_Tools:
  infrastructure_as_code: "Terraform >= 1.0"
  configuration_management: "Ansible >= 2.9"
  container_orchestration: "Docker"
  ci_cd: "GitHub Actions"

Operating_Systems:
  primary: "Ubuntu 18.04 LTS"
  planned_upgrade: "Ubuntu 20.04 LTS"
  package_manager: "apt"

Web_Stack:
  web_server: "Apache2"
  application_server: "mod_wsgi"
  database: "MySQL 8.0"
  caching: "Redis"

Monitoring:
  metrics: "Prometheus"
  visualization: "Grafana"
  logging: "ELK Stack"
  alerting: "AlertManager"
```

## 🏗️ Organizational Conventions

### Naming Conventions
```yaml
# Naming Standards
Resources:
  terraform_resources: "snake_case"
  openstack_instances: "web[number]" # web1, web2, web3
  security_groups: "[environment]-[service]-[protocol]" # prod-web-http
  networks: "[environment]-[purpose]-network" # prod-frontend-network

Files:
  terraform: "terraform-[purpose].tf" # terraform-instances.tf
  ansible_playbooks: "[purpose].yml" # site-servers-setup-all.yml
  ansible_roles: "[category]/[service]" # web/apache2, system/docker

Variables:
  terraform: "snake_case" # unique_network_name
  ansible: "snake_case" # apache_port
  environment: "UPPER_CASE" # OPENSTACK_AUTH_URL

Git:
  branches: "feature/[description]" # feature/add-monitoring
  commits: "type: description" # feat: add load balancer configuration
  tags: "v[major].[minor].[patch]" # v1.2.3
```

### Security Policies
```yaml
# Security Guidelines
Access_Control:
  ssh_access: "key-based only, no passwords"
  sudo_access: "specific users only"
  service_accounts: "minimal permissions principle"

Network_Security:
  default_policy: "deny all, allow specific"
  ssh_port: "custom port (not 22)"
  internal_communication: "private networks only"
  external_access: "load balancer only"

Data_Protection:
  encryption_at_rest: "required for databases"
  encryption_in_transit: "TLS 1.2+ required"
  backup_encryption: "required"
  secret_management: "Ansible Vault + environment variables"

Compliance:
  log_retention: "90 days minimum"
  access_logging: "all administrative actions"
  change_management: "all changes via git"
  vulnerability_scanning: "weekly automated scans"
```

## 🔧 Technical Patterns and Standards

### Terraform Patterns
```hcl
# Standard Resource Definition Pattern
resource "openstack_compute_instance_v2" "web_server" {
  name            = "${var.environment}-web-${count.index + 1}"
  image_name      = var.image_name
  flavor_id       = var.flavor_id
  key_pair        = openstack_compute_keypair_v2.terraform_key.name
  security_groups = [
    "default",
    openstack_compute_secgroup_v2.web_security_group.name
  ]
  
  network {
    name = var.network_name
  }
  
  metadata = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
  
  tags = [
    "web-server",
    var.environment
  ]
}
```

### Ansible Patterns
```yaml
# Standard Playbook Structure
---
- name: Configure web servers
  hosts: web_servers
  become: yes
  vars:
    apache_port: 80
    apache_ssl_port: 443
  
  pre_tasks:
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      tags: ['packages']
  
  roles:
    - role: system/common
      tags: ['system']
    - role: web/apache2
      tags: ['web']
    - role: monitoring/node_exporter
      tags: ['monitoring']
```

## 📊 Common Workflows and Procedures

### Infrastructure Changes
```bash
# Standard Infrastructure Change Workflow

# 1. Planning Phase
terraform plan -var-file="environments/${ENV}.tfvars" -out=tfplan

# 2. Apply Phase
terraform apply tfplan

# 3. Configuration Phase
ansible-playbook -i inventory/${ENV}/hosts playbook.yml --limit=new_servers

# 4. Validation Phase
# - Test connectivity
# - Verify services
# - Check monitoring
# - Validate security
```

This knowledge base helps Devin understand your specific infrastructure patterns, conventions, and requirements for effective DevOps collaboration.
