# CI/CD Integration with Devin

Streamline your continuous integration and deployment processes by leveraging Devin's expertise in pipeline automation, testing strategies, and deployment orchestration for your OpenStack infrastructure.

## 🎯 CI/CD Pipeline Design with Devin

### Complete CI/CD Architecture
```
CI/CD Pipeline Flow:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Code Commit   │───►│   Build & Test  │───►│   Deploy Stage  │
│   (GitHub)      │    │   (GitHub Actions)   │   (Staging)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │   Quality Gates │    │   Deploy Prod   │
                       │   (Tests/Scans) │───►│   (Production)  │
                       └─────────────────┘    └─────────────────┘
```

### Devin-Assisted CI/CD Implementation
```
Request: "Help me set up a complete CI/CD pipeline for our infrastructure code"

Devin's Approach:
1. Analyze current repository structure and workflows
2. Design multi-stage pipeline architecture
3. Create GitHub Actions workflows for different environments
4. Implement testing and validation stages
5. Set up deployment automation with rollback capabilities
6. Configure monitoring and notification integration
7. Establish security scanning and compliance checks
```

## 🚀 GitHub Actions Workflows

### Infrastructure CI Pipeline
```yaml
# .github/workflows/infrastructure-ci.yml
name: Infrastructure CI

on:
  push:
    branches: [ main, develop ]
    paths: 
      - 'terraform/**'
      - 'ansible/**'
      - '.github/workflows/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'terraform/**'
      - 'ansible/**'

env:
  TF_VERSION: '1.5.0'
  ANSIBLE_VERSION: '2.14'

jobs:
  terraform-validate:
    name: Terraform Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: ./

      - name: Terraform Init
        run: terraform init -backend=false

      - name: Terraform Validate
        run: terraform validate

      - name: Run tfsec Security Scan
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          soft_fail: true

  ansible-validate:
    name: Ansible Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install Ansible and dependencies
        run: |
          pip install ansible==${{ env.ANSIBLE_VERSION }}
          pip install ansible-lint yamllint

      - name: Run YAML Lint
        run: yamllint .

      - name: Run Ansible Lint
        run: ansible-lint playbook.yml roles/

      - name: Ansible Syntax Check
        run: ansible-playbook --syntax-check playbook.yml

  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform,ansible
          output_format: sarif
          output_file_path: reports/results.sarif

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: reports/results.sarif

  integration-test:
    name: Integration Testing
    runs-on: ubuntu-latest
    needs: [terraform-validate, ansible-validate]
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Plan
        run: |
          terraform init
          terraform plan -var-file="environments/staging.tfvars" -out=tfplan
        env:
          TF_VAR_auth_url: ${{ secrets.OPENSTACK_AUTH_URL }}
          TF_VAR_user_name: ${{ secrets.OPENSTACK_USERNAME }}
          TF_VAR_password: ${{ secrets.OPENSTACK_PASSWORD }}
          TF_VAR_tenant_name: ${{ secrets.OPENSTACK_PROJECT_NAME }}

      - name: Upload Terraform Plan
        uses: actions/upload-artifact@v3
        with:
          name: terraform-plan
          path: tfplan
```

### Deployment Pipeline
```yaml
# .github/workflows/deploy.yml
name: Deploy Infrastructure

on:
  push:
    branches: [ main ]
    paths:
      - 'terraform/**'
      - 'ansible/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging
    if: github.ref == 'refs/heads/main' || github.event.inputs.environment == 'staging'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.5.0'

      - name: Setup Python for Ansible
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install Ansible
        run: pip install ansible

      - name: Deploy Infrastructure
        run: |
          # Initialize and apply Terraform
          terraform init
          terraform workspace select staging || terraform workspace new staging
          terraform plan -var-file="environments/staging.tfvars" -out=tfplan
          terraform apply tfplan
          
          # Get infrastructure outputs for Ansible
          terraform output -json > terraform_outputs.json
        env:
          TF_VAR_auth_url: ${{ secrets.OPENSTACK_AUTH_URL }}
          TF_VAR_user_name: ${{ secrets.OPENSTACK_USERNAME }}
          TF_VAR_password: ${{ secrets.OPENSTACK_PASSWORD }}
          TF_VAR_tenant_name: ${{ secrets.OPENSTACK_PROJECT_NAME }}

      - name: Configure Infrastructure
        run: |
          # Generate dynamic inventory from Terraform outputs
          python scripts/generate_inventory.py terraform_outputs.json > inventory/staging/hosts
          
          # Run Ansible playbook
          ansible-playbook -i inventory/staging/hosts playbook.yml
        env:
          ANSIBLE_HOST_KEY_CHECKING: False

      - name: Run Health Checks
        run: |
          python scripts/health_check.py --environment staging

      - name: Notify Deployment Status
        uses: 8398a7/action-slack@v3
        if: always()
        with:
          status: ${{ job.status }}
          channel: '#devops'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' && success()
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.5.0'

      - name: Deploy to Production
        run: |
          terraform init
          terraform workspace select production || terraform workspace new production
          terraform plan -var-file="environments/production.tfvars" -out=tfplan
          terraform apply tfplan
        env:
          TF_VAR_auth_url: ${{ secrets.OPENSTACK_AUTH_URL }}
          TF_VAR_user_name: ${{ secrets.OPENSTACK_USERNAME }}
          TF_VAR_password: ${{ secrets.OPENSTACK_PASSWORD }}
          TF_VAR_tenant_name: ${{ secrets.OPENSTACK_PROJECT_NAME }}

      - name: Production Health Checks
        run: |
          python scripts/health_check.py --environment production
          python scripts/smoke_tests.py --environment production
```

## 🔧 Testing and Validation

### Infrastructure Testing Framework
```python
# scripts/health_check.py
#!/usr/bin/env python3
"""
Infrastructure health check script for CI/CD pipeline
"""

import argparse
import json
import requests
import subprocess
import sys
import time
from typing import Dict, List

class HealthChecker:
    def __init__(self, environment: str):
        self.environment = environment
        self.results = []

    def check_terraform_state(self) -> bool:
        """Verify Terraform state is consistent"""
        try:
            result = subprocess.run(
                ['terraform', 'plan', '-detailed-exitcode'],
                capture_output=True,
                text=True
            )
            # Exit code 0 = no changes, 2 = changes detected
            if result.returncode in [0, 2]:
                self.results.append({"test": "terraform_state", "status": "pass"})
                return True
            else:
                self.results.append({
                    "test": "terraform_state", 
                    "status": "fail",
                    "error": result.stderr
                })
                return False
        except Exception as e:
            self.results.append({
                "test": "terraform_state",
                "status": "fail", 
                "error": str(e)
            })
            return False

    def check_web_servers(self, servers: List[str]) -> bool:
        """Check web server availability"""
        all_healthy = True
        for server in servers:
            try:
                response = requests.get(f"http://{server}", timeout=10)
                if response.status_code == 200:
                    self.results.append({
                        "test": f"web_server_{server}",
                        "status": "pass"
                    })
                else:
                    self.results.append({
                        "test": f"web_server_{server}",
                        "status": "fail",
                        "error": f"HTTP {response.status_code}"
                    })
                    all_healthy = False
            except Exception as e:
                self.results.append({
                    "test": f"web_server_{server}",
                    "status": "fail",
                    "error": str(e)
                })
                all_healthy = False
        return all_healthy

    def check_ssh_connectivity(self, servers: List[str]) -> bool:
        """Check SSH connectivity to servers"""
        all_reachable = True
        for server in servers:
            try:
                result = subprocess.run(
                    ['ssh', '-o', 'ConnectTimeout=10', '-o', 'BatchMode=yes',
                     f'ubuntu@{server}', 'echo "SSH OK"'],
                    capture_output=True,
                    text=True,
                    timeout=15
                )
                if result.returncode == 0:
                    self.results.append({
                        "test": f"ssh_{server}",
                        "status": "pass"
                    })
                else:
                    self.results.append({
                        "test": f"ssh_{server}",
                        "status": "fail",
                        "error": result.stderr
                    })
                    all_reachable = False
            except Exception as e:
                self.results.append({
                    "test": f"ssh_{server}",
                    "status": "fail",
                    "error": str(e)
                })
                all_reachable = False
        return all_reachable

    def run_all_checks(self) -> bool:
        """Run all health checks"""
        print(f"Running health checks for {self.environment} environment...")
        
        # Get server IPs from Terraform output
        try:
            result = subprocess.run(
                ['terraform', 'output', '-json'],
                capture_output=True,
                text=True
            )
            outputs = json.loads(result.stdout)
            web_servers = outputs.get('web_server_ips', {}).get('value', [])
        except Exception as e:
            print(f"Failed to get Terraform outputs: {e}")
            return False

        # Run checks
        checks = [
            self.check_terraform_state(),
            self.check_ssh_connectivity(web_servers),
            self.check_web_servers(web_servers)
        ]

        # Print results
        passed = sum(1 for result in self.results if result['status'] == 'pass')
        total = len(self.results)
        
        print(f"\nHealth Check Results: {passed}/{total} passed")
        for result in self.results:
            status_icon = "✅" if result['status'] == 'pass' else "❌"
            print(f"{status_icon} {result['test']}")
            if result['status'] == 'fail' and 'error' in result:
                print(f"   Error: {result['error']}")

        return all(checks)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Infrastructure health checker')
    parser.add_argument('--environment', required=True, 
                       choices=['staging', 'production'],
                       help='Environment to check')
    
    args = parser.parse_args()
    
    checker = HealthChecker(args.environment)
    success = checker.run_all_checks()
    
    sys.exit(0 if success else 1)
```

### Smoke Tests
```python
# scripts/smoke_tests.py
#!/usr/bin/env python3
"""
Smoke tests for deployed infrastructure
"""

import requests
import subprocess
import json
import time
import sys

def test_load_balancer_distribution():
    """Test that load balancer distributes requests"""
    # Implementation for load balancer testing
    pass

def test_database_connectivity():
    """Test database connectivity and basic operations"""
    # Implementation for database testing
    pass

def test_monitoring_endpoints():
    """Test that monitoring endpoints are accessible"""
    endpoints = [
        "http://monitoring-server:9090/api/v1/query?query=up",  # Prometheus
        "http://monitoring-server:3000/api/health",             # Grafana
    ]
    
    for endpoint in endpoints:
        try:
            response = requests.get(endpoint, timeout=10)
            assert response.status_code == 200
            print(f"✅ {endpoint} - OK")
        except Exception as e:
            print(f"❌ {endpoint} - FAILED: {e}")
            return False
    return True

if __name__ == "__main__":
    tests = [
        test_monitoring_endpoints,
        test_load_balancer_distribution,
        test_database_connectivity
    ]
    
    failed_tests = []
    for test in tests:
        try:
            if not test():
                failed_tests.append(test.__name__)
        except Exception as e:
            print(f"❌ {test.__name__} - ERROR: {e}")
            failed_tests.append(test.__name__)
    
    if failed_tests:
        print(f"\n❌ {len(failed_tests)} tests failed: {', '.join(failed_tests)}")
        sys.exit(1)
    else:
        print(f"\n✅ All smoke tests passed!")
        sys.exit(0)
```

## 🛡️ Security and Compliance Integration

### Security Scanning Pipeline
```yaml
# .github/workflows/security-scan.yml
name: Security Scanning

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    name: Security Analysis
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/terraform

      - name: Infrastructure Security Scan
        run: |
          # Custom security checks
          python scripts/security_audit.py
```

## 🎯 CI/CD Best Practices with Devin

### Pipeline Optimization Strategies
```
Devin Workflow for CI/CD Enhancement:

1. Pipeline Analysis
   "Devin, analyze our current CI/CD pipeline and suggest optimizations"
   
2. Performance Improvement
   "Devin, help reduce our pipeline execution time while maintaining quality"
   
3. Security Integration
   "Devin, integrate comprehensive security scanning into our pipeline"
   
4. Monitoring Setup
   "Devin, add pipeline monitoring and alerting for failed deployments"
   
5. Rollback Automation
   "Devin, implement automated rollback for failed deployments"
```

### Common CI/CD Requests
```
Pipeline Setup:
"Create a complete CI/CD pipeline for our Terraform and Ansible code"

Testing Integration:
"Add comprehensive testing stages including security scans and health checks"

Multi-Environment Deployment:
"Set up automated deployment to staging and production environments"

Rollback Capabilities:
"Implement automated rollback for failed deployments"

Monitoring Integration:
"Integrate deployment monitoring and notification systems"

Performance Optimization:
"Optimize our CI/CD pipeline for faster execution and better reliability"
```

This comprehensive CI/CD integration with Devin ensures reliable, secure, and efficient deployment processes for your OpenStack infrastructure.
