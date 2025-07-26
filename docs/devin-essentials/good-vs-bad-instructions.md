# Good vs. Bad Instructions for DevOps

Learn from real examples of effective and ineffective instructions when working with Devin on DevOps tasks. Understanding these patterns will help you get better results faster.

## 🎯 Infrastructure Provisioning Examples

### ❌ Bad: Vague Infrastructure Request
```
"Set up some servers for our web application"
```

**Problems:**
- No specification of cloud provider or platform
- Missing requirements for server specifications
- No networking or security considerations
- Unclear about quantity and configuration

### ✅ Good: Specific Infrastructure Request
```
"Create 3 web servers in our OpenStack environment using Terraform:

Infrastructure requirements:
- Use Ubuntu-20.04-Latest image
- Flavor: m1.medium (flavor_id: 2)
- Attach to existing network: ${var.unique_network_name}
- Use existing security group: ${openstack_compute_secgroup_v2.sec.name}
- SSH key: ${openstack_compute_keypair_v2.terraform-key.name}

Naming convention:
- web-prod-01, web-prod-02, web-prod-03

Follow the pattern established in terraform-instances.tf but update for production use.

Validation: Ensure `terraform plan` shows 3 new instances and `terraform apply` completes without errors."
```

**Why this works:**
- Specific technical requirements
- References existing patterns and files
- Clear naming convention
- Includes validation steps

## 🔧 Configuration Management Examples

### ❌ Bad: Unclear Configuration Task
```
"Fix the Ansible playbook"
```

**Problems:**
- No description of what's broken
- Missing context about the issue
- No specific requirements for the fix

### ✅ Good: Detailed Configuration Task
```
"Update the Ansible playbook to resolve the following issue:

Problem: The nginx role is failing because it's trying to install nginx on Ubuntu 20.04 but using an outdated package repository.

Current state:
- playbook.yml includes web/nginx role
- Role is located in roles/web/nginx/
- Error occurs during package installation task

Required fix:
1. Update the nginx role to use the official nginx repository for Ubuntu 20.04
2. Ensure the role works with both Ubuntu 18.04 and 20.04
3. Add proper error handling for package installation
4. Update any deprecated Ansible syntax (we're using Ansible 2.9+)

Testing:
- Run `ansible-playbook --check playbook.yml` to validate syntax
- Test on a staging server with Ubuntu 20.04
- Verify nginx starts and serves the default page

Files to check: roles/web/nginx/tasks/main.yml, roles/web/nginx/vars/main.yml"
```

**Why this works:**
- Describes the specific problem
- Provides context about current state
- Lists exact requirements for the solution
- Includes testing and validation steps
- References specific files to examine

## 🚀 CI/CD Pipeline Examples

### ❌ Bad: Generic Pipeline Request
```
"Create a CI/CD pipeline"
```

**Problems:**
- No specification of platform (GitHub Actions, Jenkins, GitLab CI)
- Missing details about what should be built/deployed
- No requirements for testing or validation

### ✅ Good: Comprehensive Pipeline Request
```
"Create a GitHub Actions workflow for this Terraform/Ansible repository:

Workflow requirements:
1. Trigger on: push to main branch and pull requests
2. Jobs needed:
   - Terraform validation (terraform fmt, validate, plan)
   - Ansible syntax checking (ansible-playbook --syntax-check)
   - Security scanning with tfsec for Terraform files
   - Documentation generation for any changes

Environment setup:
- Use Ubuntu latest runner
- Install Terraform 1.5+, Ansible 2.9+
- Configure OpenStack credentials from repository secrets

Workflow file location: .github/workflows/infrastructure-ci.yml

Success criteria:
- All checks pass for valid Terraform/Ansible code
- Pipeline fails appropriately for syntax errors
- Secrets are handled securely
- Workflow completes in under 10 minutes

Reference our existing terraform-instances.tf and playbook.yml for context."
```

**Why this works:**
- Specifies exact platform and requirements
- Lists all necessary jobs and tools
- Includes security considerations
- Provides clear success criteria
- References existing code for context

## 🔍 Monitoring and Observability Examples

### ❌ Bad: Vague Monitoring Request
```
"Add monitoring to our infrastructure"
```

**Problems:**
- No specification of monitoring tools
- Missing details about what to monitor
- No alerting requirements

### ✅ Good: Detailed Monitoring Setup
```
"Implement comprehensive monitoring for our OpenStack web servers:

Monitoring stack:
- Prometheus for metrics collection
- Grafana for visualization
- AlertManager for notifications

Infrastructure to monitor:
- Existing web1 and web2 servers (from terraform-instances.tf)
- System metrics: CPU, memory, disk, network
- Application metrics: HTTP response times, error rates
- Infrastructure metrics: OpenStack resource usage

Implementation approach:
1. Add monitoring server to Terraform configuration
   - Instance name: monitoring-server
   - Flavor: m1.large (needs more resources)
   - Security group: allow 9090 (Prometheus), 3000 (Grafana)

2. Create Ansible roles:
   - monitoring/prometheus (server setup)
   - monitoring/node-exporter (install on web servers)
   - monitoring/grafana (dashboard setup)

3. Update existing web server configuration:
   - Install node_exporter on web1 and web2
   - Configure firewall to allow monitoring access

Alerting rules:
- CPU usage > 80% for 5 minutes
- Memory usage > 90% for 5 minutes
- Disk space < 10% remaining
- HTTP service down

Validation:
- Prometheus targets healthy at http://[monitoring-ip]:9090/targets
- Grafana dashboards accessible at http://[monitoring-ip]:3000
- Test alerts by simulating high load
- Verify all web servers appear in monitoring"
```

**Why this works:**
- Specifies exact tools and architecture
- Details what metrics to collect
- Provides implementation roadmap
- Includes specific alerting thresholds
- Lists comprehensive validation steps

## 🛡️ Security Configuration Examples

### ❌ Bad: Generic Security Request
```
"Make our infrastructure more secure"
```

**Problems:**
- No specific security requirements
- Missing compliance or regulatory context
- No prioritization of security measures

### ✅ Good: Targeted Security Implementation
```
"Implement security hardening for our OpenStack web servers:

Security requirements:
1. SSH hardening:
   - Disable password authentication (key-only)
   - Change SSH port from 22 to 2222
   - Implement fail2ban for SSH protection

2. Firewall configuration:
   - Use UFW (Uncomplicated Firewall)
   - Allow only necessary ports: 80, 443, 2222
   - Block all other inbound traffic
   - Allow outbound for package updates

3. System hardening:
   - Automatic security updates
   - Remove unnecessary packages
   - Set proper file permissions on sensitive files

Implementation:
- Update existing Ansible roles in playbook.yml
- Modify security group rules in terraform-networks.tf
- Add new role: system/security-hardening

Current infrastructure context:
- 2 Ubuntu 18.04 web servers (web1, web2)
- Apache2 web server running
- SSH access currently on port 22
- Basic security group allowing HTTP/HTTPS

Testing plan:
1. Test SSH access on new port 2222
2. Verify web services still accessible on 80/443
3. Confirm fail2ban is blocking repeated failed attempts
4. Validate firewall rules with `ufw status`

Rollback plan:
- Keep original SSH port open during testing
- Document all changes for easy reversal
- Test on staging environment first"
```

**Why this works:**
- Specific security measures listed
- Implementation approach clearly defined
- Considers existing infrastructure
- Includes comprehensive testing
- Provides rollback strategy

## 🎯 Key Takeaways

### Always Include:
1. **Specific technical requirements**
2. **Context about existing infrastructure**
3. **Clear validation/testing steps**
4. **References to existing files and patterns**
5. **Security and operational considerations**

### Avoid:
1. **Vague or generic requests**
2. **Missing context about current state**
3. **No validation criteria**
4. **Unclear success metrics**
5. **Ignoring existing patterns and conventions**

### Pro Tips:
- Reference specific files in your repository
- Include expected outcomes and success criteria
- Provide rollback plans for critical changes
- Consider security implications
- Think about integration with existing systems

Remember: Good instructions lead to better results, fewer iterations, and more reliable infrastructure automation.
