# Infrastructure Monitoring with Devin

Implement comprehensive monitoring solutions for your OpenStack infrastructure using Devin's expertise in monitoring tools, alerting systems, and observability best practices.

## 🎯 Monitoring Stack Design with Devin

### Complete Monitoring Architecture
```
Monitoring Stack Components:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Prometheus    │    │     Grafana     │    │  AlertManager   │
│   (Metrics)     │◄──►│ (Visualization) │    │  (Alerting)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                        ▲                        ▲
         │                        │                        │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Node Exporters  │    │   Log Agents    │    │   Notification  │
│ (System Metrics)│    │   (ELK Stack)   │    │   Channels      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Devin-Assisted Monitoring Implementation
```
Request: "Help me implement comprehensive monitoring for our OpenStack infrastructure"

Devin's Approach:
1. Analyze current infrastructure (web1, web2, network setup)
2. Design monitoring architecture
3. Create Terraform configuration for monitoring server
4. Develop Ansible playbooks for monitoring stack
5. Configure dashboards and alerting rules
6. Set up log aggregation and analysis
7. Implement health checks and synthetic monitoring
```

## 🚀 Prometheus Configuration

### Terraform Configuration for Monitoring Server
```hcl
# monitoring-infrastructure.tf
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
  
  metadata = {
    Environment = var.environment
    Service     = "monitoring"
    ManagedBy   = "terraform"
  }
  
  tags = ["monitoring", "prometheus", "grafana"]
}

resource "openstack_compute_secgroup_v2" "monitoring_sg" {
  name        = "${var.environment}-monitoring-sg"
  description = "Security group for monitoring services"
}

# Prometheus
resource "openstack_compute_secgroup_rule_v2" "prometheus" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 9090
  port_range_max    = 9090
  remote_ip_prefix  = "10.0.0.0/8"
  security_group_id = openstack_compute_secgroup_v2.monitoring_sg.id
}

# Grafana
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

### Prometheus Configuration
```yaml
# roles/monitoring/prometheus/templates/prometheus.yml.j2
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: '{{ environment }}'
    region: '{{ ansible_region | default("unknown") }}'

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'localhost:9093'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets:
{% for host in groups['web_servers'] %}
        - '{{ hostvars[host]['ansible_default_ipv4']['address'] }}:9100'
{% endfor %}
{% for host in groups['database_servers'] %}
        - '{{ hostvars[host]['ansible_default_ipv4']['address'] }}:9100'
{% endfor %}

  - job_name: 'apache-exporter'
    static_configs:
      - targets:
{% for host in groups['web_servers'] %}
        - '{{ hostvars[host]['ansible_default_ipv4']['address'] }}:9117'
{% endfor %}

  - job_name: 'mysql-exporter'
    static_configs:
      - targets:
{% for host in groups['database_servers'] %}
        - '{{ hostvars[host]['ansible_default_ipv4']['address'] }}:9104'
{% endfor %}
```

## 📊 Grafana Dashboards

### Infrastructure Overview Dashboard
```json
{
  "dashboard": {
    "id": null,
    "title": "OpenStack Infrastructure Overview",
    "tags": ["infrastructure", "openstack"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "System Load",
        "type": "graph",
        "targets": [
          {
            "expr": "node_load1",
            "legendFormat": "{{ instance }} - 1m load"
          }
        ],
        "yAxes": [
          {
            "label": "Load",
            "min": 0
          }
        ]
      },
      {
        "id": 2,
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)",
            "legendFormat": "{{ instance }} - Memory Usage %"
          }
        ]
      },
      {
        "id": 3,
        "title": "Disk Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (node_filesystem_avail_bytes{mountpoint=\"/\"} / node_filesystem_size_bytes{mountpoint=\"/\"} * 100)",
            "legendFormat": "{{ instance }} - Root Disk Usage %"
          }
        ]
      }
    ]
  }
}
```

## 🔔 Alerting Rules

### Critical Infrastructure Alerts
```yaml
# /etc/prometheus/rules/infrastructure.yml
groups:
  - name: infrastructure.rules
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} has been down for more than 5 minutes."

      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for more than 10 minutes."

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 90% for more than 5 minutes."

      - alert: DiskSpaceLow
        expr: (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"} * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk usage is above 85% on root filesystem."

      - alert: WebServiceDown
        expr: up{job="apache-exporter"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Web service down on {{ $labels.instance }}"
          description: "Apache web service has been down for more than 2 minutes."
```

## 🛡️ Security Monitoring

### Security-Focused Metrics
```yaml
# Security monitoring configuration
security_monitoring:
  failed_logins:
    metric: "node_systemd_unit_state{name='ssh.service'}"
    threshold: 10
    timeframe: "5m"
  
  unauthorized_access:
    log_pattern: "authentication failure"
    alert_threshold: 5
    timeframe: "1m"
  
  privilege_escalation:
    log_pattern: "sudo.*COMMAND"
    monitor: true
    alert_on_unusual_activity: true
  
  file_integrity:
    monitored_paths:
      - "/etc/passwd"
      - "/etc/shadow"
      - "/etc/ssh/sshd_config"
      - "/etc/sudoers"
    check_interval: "1h"
```

## 📈 Performance Monitoring

### Application Performance Metrics
```yaml
# Application monitoring configuration
application_monitoring:
  web_servers:
    response_time:
      target: "< 500ms"
      critical: "> 2s"
    
    error_rate:
      target: "< 1%"
      critical: "> 5%"
    
    throughput:
      baseline: "100 req/s"
      capacity: "500 req/s"
  
  database:
    query_time:
      target: "< 100ms"
      critical: "> 1s"
    
    connections:
      max: 100
      warning: 80
    
    replication_lag:
      target: "< 1s"
      critical: "> 10s"
```

## 🎯 Monitoring Best Practices with Devin

### Automated Monitoring Setup
```
Devin Workflow for Monitoring Implementation:

1. Infrastructure Analysis
   "Devin, analyze our current infrastructure and recommend monitoring strategy"
   
2. Monitoring Stack Deployment
   "Devin, deploy Prometheus, Grafana, and AlertManager to our monitoring server"
   
3. Metrics Collection Setup
   "Devin, configure node exporters on all servers and set up service monitoring"
   
4. Dashboard Creation
   "Devin, create comprehensive Grafana dashboards for our infrastructure"
   
5. Alerting Configuration
   "Devin, set up alerting rules for critical infrastructure events"
   
6. Integration Setup
   "Devin, integrate monitoring with our Slack/email notification systems"
```

### Common Monitoring Requests
```
Infrastructure Health:
"Set up monitoring for all OpenStack instances with CPU, memory, and disk alerts"

Application Performance:
"Monitor Apache response times and error rates with appropriate thresholds"

Security Monitoring:
"Implement security monitoring for failed logins and unauthorized access attempts"

Capacity Planning:
"Set up monitoring to track resource usage trends for capacity planning"

Custom Metrics:
"Create custom metrics for our application-specific KPIs"
```

This comprehensive monitoring approach with Devin ensures complete visibility into your OpenStack infrastructure health, performance, and security.
