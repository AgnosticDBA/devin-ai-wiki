# Deployment Capabilities with Devin

Leverage Devin's advanced deployment capabilities to automate, orchestrate, and manage your infrastructure deployments across multiple environments with confidence and reliability.

## 🎯 Deployment Architecture Overview

### Multi-Environment Deployment Strategy
```
Deployment Pipeline Architecture:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Development   │───►│     Staging     │───►│   Production    │
│   Environment   │    │   Environment   │    │   Environment   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                        ▲                        ▲
         │                        │                        │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Feature       │    │   Integration   │    │   Blue/Green    │
│   Testing       │    │   Testing       │    │   Deployment    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Devin's Deployment Capabilities
```
Core Deployment Features:
1. Infrastructure Provisioning (Terraform)
2. Configuration Management (Ansible)
3. Application Deployment
4. Database Migrations
5. Health Checks and Validation
6. Rollback and Recovery
7. Monitoring Integration
8. Notification and Reporting
```

## 🚀 Infrastructure Deployment

### Terraform-Based Infrastructure Deployment
```hcl
# environments/production/main.tf
terraform {
  required_version = ">= 1.0"
  
  backend "swift" {
    container         = "terraform-state-prod"
    archive_container = "terraform-state-archive-prod"
  }
  
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 1.48.0"
    }
  }
}

module "infrastructure" {
  source = "../../modules/openstack-infrastructure"
  
  environment = "production"
  
  # Instance Configuration
  web_server_count    = 3
  web_server_flavor   = "m1.large"
  db_server_flavor    = "m1.xlarge"
  
  # Network Configuration
  network_cidr        = "10.1.0.0/16"
  public_subnet_cidr  = "10.1.1.0/24"
  private_subnet_cidr = "10.1.2.0/24"
  
  # Security Configuration
  allowed_ssh_cidrs   = ["10.0.0.0/8"]
  allowed_web_cidrs   = ["0.0.0.0/0"]
  
  # Monitoring Configuration
  enable_monitoring   = true
  monitoring_flavor   = "m1.large"
  
  # Backup Configuration
  enable_backups      = true
  backup_retention    = 30
  
  tags = {
    Environment = "production"
    Project     = "devops101"
    ManagedBy   = "terraform"
    Owner       = "devops-team"
  }
}

# Output important values for other tools
output "web_server_ips" {
  description = "IP addresses of web servers"
  value       = module.infrastructure.web_server_ips
}

output "load_balancer_ip" {
  description = "Load balancer IP address"
  value       = module.infrastructure.load_balancer_ip
}

output "database_endpoint" {
  description = "Database connection endpoint"
  value       = module.infrastructure.database_endpoint
  sensitive   = true
}
```

### Automated Deployment Script
```bash
#!/bin/bash
# scripts/deploy.sh - Comprehensive deployment script

set -euo pipefail

# Configuration
ENVIRONMENT=${1:-staging}
DRY_RUN=${2:-false}
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging functions
log_info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Validation functions
validate_environment() {
    if [[ ! "$ENVIRONMENT" =~ ^(development|staging|production)$ ]]; then
        log_error "Invalid environment: $ENVIRONMENT"
        log_info "Valid environments: development, staging, production"
        exit 1
    fi
}

validate_prerequisites() {
    log_info "Validating prerequisites..."
    
    # Check required tools
    for tool in terraform ansible python3; do
        if ! command -v "$tool" &> /dev/null; then
            log_error "$tool is not installed or not in PATH"
            exit 1
        fi
    done
    
    # Check environment variables
    required_vars=("TF_VAR_auth_url" "TF_VAR_user_name" "TF_VAR_password" "TF_VAR_tenant_name")
    for var in "${required_vars[@]}"; do
        if [[ -z "${!var:-}" ]]; then
            log_error "Required environment variable $var is not set"
            exit 1
        fi
    done
    
    log_success "Prerequisites validated"
}

# Terraform functions
terraform_init() {
    log_info "Initializing Terraform..."
    cd "$PROJECT_ROOT"
    
    terraform init \
        -backend-config="container=terraform-state-$ENVIRONMENT" \
        -backend-config="archive_container=terraform-state-archive-$ENVIRONMENT"
    
    terraform workspace select "$ENVIRONMENT" || terraform workspace new "$ENVIRONMENT"
    log_success "Terraform initialized"
}

terraform_plan() {
    log_info "Creating Terraform plan..."
    
    terraform plan \
        -var-file="environments/$ENVIRONMENT.tfvars" \
        -out="$ENVIRONMENT.tfplan" \
        -detailed-exitcode
    
    local exit_code=$?
    
    case $exit_code in
        0)
            log_info "No changes detected in Terraform plan"
            return 0
            ;;
        1)
            log_error "Terraform plan failed"
            return 1
            ;;
        2)
            log_info "Changes detected in Terraform plan"
            return 2
            ;;
    esac
}

terraform_apply() {
    log_info "Applying Terraform changes..."
    
    if [[ "$DRY_RUN" == "true" ]]; then
        log_warning "DRY RUN: Would apply Terraform plan"
        return 0
    fi
    
    terraform apply "$ENVIRONMENT.tfplan"
    log_success "Terraform changes applied"
}

# Ansible functions
ansible_configure() {
    log_info "Configuring infrastructure with Ansible..."
    
    # Generate dynamic inventory from Terraform outputs
    terraform output -json > terraform_outputs.json
    python3 scripts/generate_inventory.py terraform_outputs.json > "inventory/$ENVIRONMENT/hosts"
    
    if [[ "$DRY_RUN" == "true" ]]; then
        log_warning "DRY RUN: Would run Ansible playbook"
        ansible-playbook \
            -i "inventory/$ENVIRONMENT/hosts" \
            --check \
            --diff \
            playbook.yml
        return 0
    fi
    
    ansible-playbook \
        -i "inventory/$ENVIRONMENT/hosts" \
        playbook.yml
    
    log_success "Infrastructure configured with Ansible"
}

# Health check functions
run_health_checks() {
    log_info "Running health checks..."
    
    if [[ "$DRY_RUN" == "true" ]]; then
        log_warning "DRY RUN: Would run health checks"
        return 0
    fi
    
    python3 scripts/health_check.py --environment "$ENVIRONMENT"
    
    if [[ $? -eq 0 ]]; then
        log_success "Health checks passed"
    else
        log_error "Health checks failed"
        return 1
    fi
}

# Notification functions
send_notification() {
    local status=$1
    local message=$2
    
    if [[ -n "${SLACK_WEBHOOK_URL:-}" ]]; then
        python3 scripts/slack_notifier.py \
            --webhook-url "$SLACK_WEBHOOK_URL" \
            --type deployment \
            --environment "$ENVIRONMENT" \
            --status "$status" \
            --message "$message"
    fi
}

# Rollback functions
rollback_deployment() {
    log_warning "Rolling back deployment..."
    
    # Get previous Terraform state
    terraform workspace select "$ENVIRONMENT"
    
    # Rollback to previous state
    # This is a simplified example - implement based on your backup strategy
    if [[ -f "backups/$ENVIRONMENT-previous.tfstate" ]]; then
        cp "backups/$ENVIRONMENT-previous.tfstate" terraform.tfstate
        terraform apply -auto-approve
        log_success "Rollback completed"
    else
        log_error "No previous state found for rollback"
        return 1
    fi
}

# Main deployment function
deploy() {
    log_info "Starting deployment to $ENVIRONMENT environment"
    
    # Send start notification
    send_notification "started" "Deployment to $ENVIRONMENT initiated"
    
    # Backup current state
    if [[ "$ENVIRONMENT" == "production" ]]; then
        mkdir -p backups
        cp terraform.tfstate "backups/$ENVIRONMENT-$(date +%Y%m%d-%H%M%S).tfstate" 2>/dev/null || true
    fi
    
    # Execute deployment steps
    if validate_prerequisites && \
       terraform_init && \
       terraform_plan; then
        
        local plan_exit_code=$?
        
        if [[ $plan_exit_code -eq 2 ]] || [[ "$DRY_RUN" == "true" ]]; then
            if terraform_apply && \
               ansible_configure && \
               run_health_checks; then
                
                log_success "Deployment to $ENVIRONMENT completed successfully"
                send_notification "success" "Deployment to $ENVIRONMENT completed successfully"
                return 0
            else
                log_error "Deployment failed"
                send_notification "failed" "Deployment to $ENVIRONMENT failed"
                
                if [[ "$ENVIRONMENT" == "production" ]]; then
                    log_warning "Production deployment failed - consider rollback"
                fi
                return 1
            fi
        else
            log_info "No changes to deploy"
            return 0
        fi
    else
        log_error "Pre-deployment validation failed"
        send_notification "failed" "Pre-deployment validation failed for $ENVIRONMENT"
        return 1
    fi
}

# Main execution
main() {
    validate_environment
    
    log_info "Deployment Configuration:"
    log_info "  Environment: $ENVIRONMENT"
    log_info "  Dry Run: $DRY_RUN"
    log_info "  Project Root: $PROJECT_ROOT"
    
    if deploy; then
        exit 0
    else
        exit 1
    fi
}

# Script execution
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

## 🔧 Application Deployment

### Container Deployment with Ansible
```yaml
# roles/application/deploy/tasks/main.yml
---
- name: Create application directory
  file:
    path: /opt/application
    state: directory
    owner: app
    group: app
    mode: '0755'
  tags: ['application', 'setup']

- name: Deploy application configuration
  template:
    src: app.conf.j2
    dest: /opt/application/app.conf
    owner: app
    group: app
    mode: '0644'
    backup: yes
  notify: restart application
  tags: ['application', 'config']

- name: Pull application image
  docker_image:
    name: "{{ application_image }}"
    tag: "{{ application_version }}"
    source: pull
    force_source: yes
  tags: ['application', 'deploy']

- name: Stop existing application container
  docker_container:
    name: "{{ application_name }}"
    state: stopped
  ignore_errors: yes
  tags: ['application', 'deploy']

- name: Remove existing application container
  docker_container:
    name: "{{ application_name }}"
    state: absent
  ignore_errors: yes
  tags: ['application', 'deploy']

- name: Deploy application container
  docker_container:
    name: "{{ application_name }}"
    image: "{{ application_image }}:{{ application_version }}"
    state: started
    restart_policy: always
    ports:
      - "{{ application_port }}:{{ application_port }}"
    env:
      DATABASE_URL: "{{ database_url }}"
      REDIS_URL: "{{ redis_url }}"
      LOG_LEVEL: "{{ log_level }}"
    volumes:
      - "/opt/application/app.conf:/app/config/app.conf:ro"
      - "/var/log/application:/app/logs"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:{{ application_port }}/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  tags: ['application', 'deploy']

- name: Wait for application to be healthy
  uri:
    url: "http://localhost:{{ application_port }}/health"
    method: GET
    status_code: 200
  retries: 10
  delay: 5
  tags: ['application', 'verify']

- name: Register application with load balancer
  uri:
    url: "{{ load_balancer_api }}/register"
    method: POST
    body_format: json
    body:
      instance: "{{ ansible_default_ipv4.address }}"
      port: "{{ application_port }}"
      health_check: "/health"
  when: load_balancer_api is defined
  tags: ['application', 'loadbalancer']
```

## 🛡️ Blue-Green Deployment

### Blue-Green Deployment Strategy
```python
# scripts/blue_green_deploy.py
#!/usr/bin/env python3
"""
Blue-Green deployment script for zero-downtime deployments
"""

import json
import subprocess
import time
import requests
from typing import Dict, List

class BlueGreenDeployer:
    def __init__(self, environment: str):
        self.environment = environment
        self.current_color = self.get_current_color()
        self.target_color = 'green' if self.current_color == 'blue' else 'blue'
        
    def get_current_color(self) -> str:
        """Determine current active deployment color"""
        try:
            result = subprocess.run(
                ['terraform', 'output', '-json', 'active_color'],
                capture_output=True,
                text=True
            )
            if result.returncode == 0:
                output = json.loads(result.stdout)
                return output.get('value', 'blue')
        except:
            pass
        return 'blue'
    
    def deploy_target_environment(self) -> bool:
        """Deploy to target color environment"""
        print(f"Deploying to {self.target_color} environment...")
        
        # Update Terraform variables for target deployment
        env_vars = {
            'TF_VAR_active_color': self.target_color,
            'TF_VAR_deploy_blue': 'true' if self.target_color == 'blue' else 'false',
            'TF_VAR_deploy_green': 'true' if self.target_color == 'green' else 'false'
        }
        
        # Apply Terraform changes
        result = subprocess.run(
            ['terraform', 'apply', '-auto-approve', 
             f'-var-file=environments/{self.environment}.tfvars'],
            env={**os.environ, **env_vars}
        )
        
        return result.returncode == 0
    
    def run_health_checks(self, instances: List[str]) -> bool:
        """Run health checks on target instances"""
        print(f"Running health checks on {self.target_color} instances...")
        
        for instance in instances:
            for attempt in range(30):  # 5 minutes of retries
                try:
                    response = requests.get(
                        f"http://{instance}/health",
                        timeout=10
                    )
                    if response.status_code == 200:
                        print(f"✓ {instance} is healthy")
                        break
                except:
                    pass
                
                if attempt == 29:
                    print(f"✗ {instance} failed health check")
                    return False
                
                time.sleep(10)
        
        return True
    
    def run_smoke_tests(self, instances: List[str]) -> bool:
        """Run smoke tests on target instances"""
        print(f"Running smoke tests on {self.target_color} instances...")
        
        # Implement your smoke tests here
        test_endpoints = ['/api/status', '/api/version', '/']
        
        for instance in instances:
            for endpoint in test_endpoints:
                try:
                    response = requests.get(
                        f"http://{instance}{endpoint}",
                        timeout=10
                    )
                    if response.status_code not in [200, 404]:  # 404 might be OK for some endpoints
                        print(f"✗ Smoke test failed for {instance}{endpoint}")
                        return False
                except Exception as e:
                    print(f"✗ Smoke test error for {instance}{endpoint}: {e}")
                    return False
        
        print("✓ All smoke tests passed")
        return True
    
    def switch_traffic(self) -> bool:
        """Switch traffic to target environment"""
        print(f"Switching traffic to {self.target_color} environment...")
        
        # Update load balancer configuration
        result = subprocess.run([
            'ansible-playbook',
            '-i', f'inventory/{self.environment}/hosts',
            'playbooks/switch_traffic.yml',
            '-e', f'target_color={self.target_color}'
        ])
        
        return result.returncode == 0
    
    def cleanup_old_environment(self) -> bool:
        """Clean up the old environment"""
        print(f"Cleaning up {self.current_color} environment...")
        
        # Scale down old environment
        env_vars = {
            'TF_VAR_deploy_blue': 'false' if self.current_color == 'blue' else 'true',
            'TF_VAR_deploy_green': 'false' if self.current_color == 'green' else 'true'
        }
        
        result = subprocess.run(
            ['terraform', 'apply', '-auto-approve',
             f'-var-file=environments/{self.environment}.tfvars'],
            env={**os.environ, **env_vars}
        )
        
        return result.returncode == 0
    
    def rollback(self) -> bool:
        """Rollback to previous environment"""
        print(f"Rolling back to {self.current_color} environment...")
        
        # Switch traffic back
        result = subprocess.run([
            'ansible-playbook',
            '-i', f'inventory/{self.environment}/hosts',
            'playbooks/switch_traffic.yml',
            '-e', f'target_color={self.current_color}'
        ])
        
        return result.returncode == 0
    
    def deploy(self) -> bool:
        """Execute blue-green deployment"""
        print(f"Starting blue-green deployment from {self.current_color} to {self.target_color}")
        
        try:
            # Step 1: Deploy to target environment
            if not self.deploy_target_environment():
                print("✗ Target environment deployment failed")
                return False
            
            # Step 2: Get target instances
            result = subprocess.run(
                ['terraform', 'output', '-json', f'{self.target_color}_instances'],
                capture_output=True,
                text=True
            )
            
            if result.returncode != 0:
                print("✗ Failed to get target instances")
                return False
            
            target_instances = json.loads(result.stdout)['value']
            
            # Step 3: Health checks
            if not self.run_health_checks(target_instances):
                print("✗ Health checks failed")
                return False
            
            # Step 4: Smoke tests
            if not self.run_smoke_tests(target_instances):
                print("✗ Smoke tests failed")
                return False
            
            # Step 5: Switch traffic
            if not self.switch_traffic():
                print("✗ Traffic switch failed")
                self.rollback()
                return False
            
            # Step 6: Final verification
            time.sleep(30)  # Allow traffic to stabilize
            if not self.run_health_checks(target_instances):
                print("✗ Post-switch health checks failed")
                self.rollback()
                return False
            
            # Step 7: Cleanup old environment
            if not self.cleanup_old_environment():
                print("⚠ Warning: Failed to cleanup old environment")
            
            print(f"✓ Blue-green deployment completed successfully")
            print(f"✓ Traffic switched from {self.current_color} to {self.target_color}")
            return True
            
        except Exception as e:
            print(f"✗ Deployment failed with error: {e}")
            self.rollback()
            return False

if __name__ == "__main__":
    import argparse
    import os
    
    parser = argparse.ArgumentParser(description='Blue-Green Deployment')
    parser.add_argument('--environment', required=True,
                       choices=['staging', 'production'],
                       help='Environment to deploy to')
    
    args = parser.parse_args()
    
    deployer = BlueGreenDeployer(args.environment)
    success = deployer.deploy()
    
    exit(0 if success else 1)
```

## 🎯 Best Practices with Devin

### Deployment Automation Strategies
```
Devin Deployment Workflows:

1. Infrastructure Deployment
   "Devin, deploy our infrastructure changes to staging environment"
   
2. Application Deployment
   "Devin, deploy version 2.1.0 of our application using blue-green strategy"
   
3. Database Migration
   "Devin, run database migrations as part of the deployment process"
   
4. Rollback Management
   "Devin, rollback the production deployment to the previous version"
   
5. Multi-Environment Promotion
   "Devin, promote the staging deployment to production"
```

### Common Deployment Requests
```
Infrastructure Deployment:
"Deploy the latest infrastructure changes to production with health checks"

Application Updates:
"Deploy application version 2.0 using zero-downtime blue-green deployment"

Configuration Changes:
"Apply the new security configurations across all environments"

Emergency Rollback:
"Immediately rollback the production deployment due to critical issues"

Scheduled Maintenance:
"Deploy the maintenance updates during the scheduled window"
```

This comprehensive deployment capability with Devin ensures reliable, automated, and scalable deployment processes for your OpenStack infrastructure and applications.
