# Terraform Automation with Devin

Leverage Devin's capabilities to enhance your Terraform workflows, from initial configuration to advanced infrastructure management. This guide provides practical examples and best practices for using Devin with Terraform in OpenStack environments.

## 🎯 Core Terraform Tasks with Devin

### Infrastructure Planning and Design
Devin excels at helping you design and plan Terraform configurations:

```
Example Request:
"I need to expand our current OpenStack infrastructure to include a load balancer 
for our web servers. Can you help me design the Terraform configuration?"

Devin's Approach:
1. Analyzes existing terraform-instances.tf and terraform-networks.tf
2. Reviews current security group configurations
3. Proposes load balancer architecture
4. Creates comprehensive Terraform configuration
5. Suggests testing and validation approach
```

### Resource Configuration Generation
```hcl
# Example: Devin-generated load balancer configuration
resource "openstack_lb_loadbalancer_v2" "web_lb" {
  name          = "${var.environment}-web-loadbalancer"
  vip_subnet_id = openstack_networking_subnet_v2.frontend_subnet.id
  
  tags = [
    "environment:${var.environment}",
    "managed-by:terraform",
    "service:web"
  ]
}

resource "openstack_lb_listener_v2" "web_listener" {
  name            = "${var.environment}-web-listener"
  protocol        = "HTTP"
  protocol_port   = 80
  loadbalancer_id = openstack_lb_loadbalancer_v2.web_lb.id
}

resource "openstack_lb_pool_v2" "web_pool" {
  name        = "${var.environment}-web-pool"
  protocol    = "HTTP"
  lb_method   = "ROUND_ROBIN"
  listener_id = openstack_lb_listener_v2.web_listener.id
}

resource "openstack_lb_member_v2" "web_members" {
  count         = length(openstack_compute_instance_v2.web_servers)
  pool_id       = openstack_lb_pool_v2.web_pool.id
  address       = openstack_compute_instance_v2.web_servers[count.index].access_ip_v4
  protocol_port = 80
  subnet_id     = openstack_networking_subnet_v2.frontend_subnet.id
}

resource "openstack_lb_monitor_v2" "web_monitor" {
  name        = "${var.environment}-web-monitor"
  pool_id     = openstack_lb_pool_v2.web_pool.id
  type        = "HTTP"
  delay       = 20
  timeout     = 10
  max_retries = 5
  url_path    = "/health"
}
```

## 🚀 Advanced Terraform Patterns

### Module Development with Devin
```
Request: "Help me create a reusable Terraform module for OpenStack web servers"

Devin's Module Structure:
modules/
└── openstack-web-server/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    └── README.md
```

#### Module Example: Web Server Module
```hcl
# modules/openstack-web-server/main.tf
resource "openstack_compute_instance_v2" "web_server" {
  count           = var.instance_count
  name            = "${var.name_prefix}-web-${count.index + 1}"
  image_name      = var.image_name
  flavor_id       = var.flavor_id
  key_pair        = var.key_pair_name
  security_groups = var.security_groups
  
  network {
    name = var.network_name
  }
  
  metadata = merge(
    var.metadata,
    {
      "terraform-module" = "openstack-web-server"
      "instance-index"   = count.index + 1
    }
  )
  
  user_data = var.user_data
  
  tags = var.tags
}

# modules/openstack-web-server/variables.tf
variable "instance_count" {
  description = "Number of web server instances to create"
  type        = number
  default     = 2
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

variable "name_prefix" {
  description = "Prefix for instance names"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.name_prefix))
    error_message = "Name prefix must contain only lowercase letters, numbers, and hyphens."
  }
}

variable "image_name" {
  description = "OpenStack image name for instances"
  type        = string
  default     = "Ubuntu-20.04-Latest"
}

variable "flavor_id" {
  description = "OpenStack flavor ID for instances"
  type        = string
  default     = "1"
}

variable "key_pair_name" {
  description = "OpenStack key pair name for SSH access"
  type        = string
}

variable "security_groups" {
  description = "List of security groups to attach to instances"
  type        = list(string)
  default     = ["default"]
}

variable "network_name" {
  description = "OpenStack network name for instances"
  type        = string
}

variable "metadata" {
  description = "Additional metadata for instances"
  type        = map(string)
  default     = {}
}

variable "user_data" {
  description = "User data script for instance initialization"
  type        = string
  default     = null
}

variable "tags" {
  description = "Tags to apply to instances"
  type        = list(string)
  default     = []
}

# modules/openstack-web-server/outputs.tf
output "instance_ids" {
  description = "IDs of created instances"
  value       = openstack_compute_instance_v2.web_server[*].id
}

output "instance_ips" {
  description = "IP addresses of created instances"
  value       = openstack_compute_instance_v2.web_server[*].access_ip_v4
}

output "instance_names" {
  description = "Names of created instances"
  value       = openstack_compute_instance_v2.web_server[*].name
}
```

### Environment-Specific Configurations
```hcl
# environments/development/main.tf
module "web_servers" {
  source = "../../modules/openstack-web-server"
  
  instance_count  = 1
  name_prefix     = "dev"
  flavor_id       = "0"  # Smaller flavor for dev
  key_pair_name   = var.ssh_key_name
  security_groups = [openstack_compute_secgroup_v2.web_sg.name]
  network_name    = var.network_name
  
  metadata = {
    Environment = "development"
    Project     = "devops101"
    Owner       = "development-team"
  }
  
  tags = ["development", "web-server"]
}

# environments/production/main.tf
module "web_servers" {
  source = "../../modules/openstack-web-server"
  
  instance_count  = 3
  name_prefix     = "prod"
  flavor_id       = "2"  # Larger flavor for prod
  key_pair_name   = var.ssh_key_name
  security_groups = [
    openstack_compute_secgroup_v2.web_sg.name,
    openstack_compute_secgroup_v2.monitoring_sg.name
  ]
  network_name    = var.network_name
  
  metadata = {
    Environment = "production"
    Project     = "devops101"
    Owner       = "operations-team"
    Backup      = "enabled"
  }
  
  tags = ["production", "web-server", "critical"]
}
```

## 🔧 Terraform Workflow Automation

### State Management with Devin
```bash
# Devin can help automate state management tasks

# Remote state configuration
terraform {
  backend "swift" {
    container         = "terraform-state"
    archive_container = "terraform-state-archive"
  }
}

# State operations
terraform state list
terraform state show openstack_compute_instance_v2.web1
terraform state mv openstack_compute_instance_v2.old_web openstack_compute_instance_v2.new_web
```

### Automated Testing and Validation
```go
// Example: Terratest integration that Devin can help create
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestOpenStackWebServers(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../",
        VarFiles:     []string{"test.tfvars"},
    }
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // Validate outputs
    instanceIPs := terraform.OutputList(t, terraformOptions, "web_server_ips")
    assert.Equal(t, 2, len(instanceIPs))
    
    // Test connectivity
    for _, ip := range instanceIPs {
        // Add connectivity tests
    }
}
```

## 📊 Infrastructure Monitoring and Observability

### Terraform-Managed Monitoring
```hcl
# Monitoring infrastructure as code
resource "openstack_compute_instance_v2" "monitoring" {
  name            = "${var.environment}-monitoring"
  image_name      = "Ubuntu-20.04-Latest"
  flavor_id       = "2"
  key_pair        = openstack_compute_keypair_v2.terraform_key.name
  security_groups = [
    "default",
    openstack_compute_secgroup_v2.monitoring_sg.name
  ]
  
  network {
    name = var.network_name
  }
  
  user_data = templatefile("${path.module}/scripts/monitoring-setup.sh", {
    prometheus_targets = join(",", openstack_compute_instance_v2.web_servers[*].access_ip_v4)
  })
}

resource "openstack_compute_secgroup_v2" "monitoring_sg" {
  name        = "${var.environment}-monitoring-sg"
  description = "Security group for monitoring services"
}

resource "openstack_compute_secgroup_rule_v2" "prometheus" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 9090
  port_range_max    = 9090
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_compute_secgroup_v2.monitoring_sg.id
}

resource "openstack_compute_secgroup_rule_v2" "grafana" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 3000
  port_range_max    = 3000
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_compute_secgroup_v2.monitoring_sg.id
}
```

## 🛡️ Security and Compliance

### Security-First Terraform Configurations
```hcl
# Security group with principle of least privilege
resource "openstack_compute_secgroup_v2" "web_sg" {
  name        = "${var.environment}-web-sg"
  description = "Security group for web servers"
}

# HTTP access
resource "openstack_compute_secgroup_rule_v2" "web_http" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 80
  port_range_max    = 80
  remote_group_id   = openstack_compute_secgroup_v2.lb_sg.id
  security_group_id = openstack_compute_secgroup_v2.web_sg.id
}

# HTTPS access
resource "openstack_compute_secgroup_rule_v2" "web_https" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 443
  port_range_max    = 443
  remote_group_id   = openstack_compute_secgroup_v2.lb_sg.id
  security_group_id = openstack_compute_secgroup_v2.web_sg.id
}

# SSH access from bastion only
resource "openstack_compute_secgroup_rule_v2" "web_ssh" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 22
  port_range_max    = 22
  remote_group_id   = openstack_compute_secgroup_v2.bastion_sg.id
  security_group_id = openstack_compute_secgroup_v2.web_sg.id
}
```

## 🎯 Best Practices with Devin

### Code Organization and Standards
```
Terraform Project Structure (Devin-recommended):

project/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── modules/
│   ├── compute/
│   ├── networking/
│   └── security/
├── scripts/
├── tests/
└── docs/
```

### Devin Workflow Integration
```
1. Planning Phase
   - "Devin, help me plan the infrastructure changes for adding a database tier"
   - Collaborative architecture design
   - Resource requirement analysis

2. Implementation Phase
   - "Devin, create the Terraform configuration for the planned database setup"
   - Module development and testing
   - Security and compliance validation

3. Deployment Phase
   - "Devin, help me deploy this to staging first, then production"
   - Automated testing and validation
   - Rollback planning and execution

4. Maintenance Phase
   - "Devin, optimize our Terraform configurations for better performance"
   - Regular security updates
   - Cost optimization recommendations
```

### Common Devin Requests for Terraform
```
Infrastructure Expansion:
"Add a Redis cache layer to our existing web infrastructure"

Security Hardening:
"Update our security groups to follow the principle of least privilege"

Performance Optimization:
"Help me implement auto-scaling for our web servers"

Disaster Recovery:
"Set up cross-region backup for our critical infrastructure"

Monitoring Integration:
"Add comprehensive monitoring to all our Terraform-managed resources"
```

This comprehensive approach to Terraform automation with Devin ensures efficient, secure, and maintainable infrastructure management for your OpenStack environment.
