# Slack Integration with Devin

Enhance your team collaboration and DevOps workflows by integrating Devin with Slack for real-time notifications, automated updates, and seamless communication.

## 🎯 Slack Integration Overview

### Core Integration Features
```
Devin's Slack Capabilities:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Deployment    │    │   Monitoring    │    │   Incident      │
│   Notifications │◄──►│   Alerts        │◄──►│   Management    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                        ▲                        ▲
         │                        │                        │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   CI/CD Status  │    │   Infrastructure│    │   Team          │
│   Updates       │    │   Changes       │    │   Collaboration │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Setting Up Slack Integration
```
Initial Setup Process:
1. Create Slack app in your workspace
2. Configure incoming webhooks
3. Set up bot permissions and scopes
4. Add webhook URLs to repository secrets
5. Configure notification channels
6. Set up automated workflows
7. Test integration functionality
```

## 🚀 Deployment Notifications

### GitHub Actions Slack Integration
```yaml
# .github/workflows/slack-notifications.yml
name: Slack Notifications

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, closed, reopened]
  workflow_run:
    workflows: ["Deploy Infrastructure"]
    types: [completed]

jobs:
  notify-deployment:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Notify Deployment Start
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              channel: '#devops',
              username: 'Devin CI/CD',
              icon_emoji: ':rocket:',
              attachments: [{
                color: 'warning',
                fields: [{
                  title: 'Deployment Started',
                  value: `Infrastructure deployment initiated for ${process.env.AS_REPO}`,
                  short: true
                }, {
                  title: 'Branch',
                  value: '${process.env.AS_REF}',
                  short: true
                }, {
                  title: 'Commit',
                  value: `<${process.env.AS_COMMIT}|${process.env.AS_COMMIT.slice(0, 8)}>`,
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  notify-pr:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: PR Notification
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              channel: '#devops',
              username: 'Devin GitHub',
              icon_emoji: ':octocat:',
              attachments: [{
                color: '${{ github.event.action == 'opened' && 'good' || github.event.pull_request.merged && 'good' || 'danger' }}',
                fields: [{
                  title: 'Pull Request ${{ github.event.action }}',
                  value: `<${{ github.event.pull_request.html_url }}|${{ github.event.pull_request.title }}>`,
                  short: false
                }, {
                  title: 'Author',
                  value: '${{ github.event.pull_request.user.login }}',
                  short: true
                }, {
                  title: 'Files Changed',
                  value: '${{ github.event.pull_request.changed_files }}',
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Infrastructure Change Notifications
```python
# scripts/slack_notifier.py
#!/usr/bin/env python3
"""
Slack notification script for infrastructure changes
"""

import json
import requests
import subprocess
import sys
from datetime import datetime
from typing import Dict, List, Optional

class SlackNotifier:
    def __init__(self, webhook_url: str, channel: str = "#devops"):
        self.webhook_url = webhook_url
        self.channel = channel

    def send_message(self, message: Dict) -> bool:
        """Send message to Slack"""
        try:
            response = requests.post(self.webhook_url, json=message)
            return response.status_code == 200
        except Exception as e:
            print(f"Failed to send Slack message: {e}")
            return False

    def notify_terraform_plan(self, plan_output: str, environment: str) -> bool:
        """Notify about Terraform plan results"""
        # Parse plan output for changes
        lines = plan_output.split('\n')
        add_count = sum(1 for line in lines if 'will be created' in line)
        change_count = sum(1 for line in lines if 'will be updated' in line)
        destroy_count = sum(1 for line in lines if 'will be destroyed' in line)

        color = 'good' if destroy_count == 0 else 'warning'
        if destroy_count > 0:
            color = 'danger'

        message = {
            "channel": self.channel,
            "username": "Devin Infrastructure",
            "icon_emoji": ":terraform:",
            "attachments": [{
                "color": color,
                "title": f"Terraform Plan - {environment.title()} Environment",
                "fields": [
                    {
                        "title": "Resources to Add",
                        "value": str(add_count),
                        "short": True
                    },
                    {
                        "title": "Resources to Change", 
                        "value": str(change_count),
                        "short": True
                    },
                    {
                        "title": "Resources to Destroy",
                        "value": str(destroy_count),
                        "short": True
                    }
                ],
                "footer": "Terraform Plan",
                "ts": int(datetime.now().timestamp())
            }]
        }

        return self.send_message(message)

    def notify_deployment_status(self, status: str, environment: str, 
                               details: Optional[str] = None) -> bool:
        """Notify about deployment status"""
        color_map = {
            'started': 'warning',
            'success': 'good', 
            'failed': 'danger',
            'rollback': 'warning'
        }

        emoji_map = {
            'started': ':rocket:',
            'success': ':white_check_mark:',
            'failed': ':x:',
            'rollback': ':leftwards_arrow_with_hook:'
        }

        message = {
            "channel": self.channel,
            "username": "Devin Deployment",
            "icon_emoji": emoji_map.get(status, ':gear:'),
            "attachments": [{
                "color": color_map.get(status, 'warning'),
                "title": f"Deployment {status.title()} - {environment.title()}",
                "text": details or f"Infrastructure deployment {status} for {environment} environment",
                "footer": "Devin CI/CD",
                "ts": int(datetime.now().timestamp())
            }]
        }

        return self.send_message(message)

    def notify_monitoring_alert(self, alert_name: str, severity: str, 
                              instance: str, description: str) -> bool:
        """Notify about monitoring alerts"""
        color_map = {
            'critical': 'danger',
            'warning': 'warning',
            'info': 'good'
        }

        message = {
            "channel": "#alerts",
            "username": "Devin Monitoring",
            "icon_emoji": ":warning:",
            "attachments": [{
                "color": color_map.get(severity.lower(), 'warning'),
                "title": f"{severity.upper()}: {alert_name}",
                "fields": [
                    {
                        "title": "Instance",
                        "value": instance,
                        "short": True
                    },
                    {
                        "title": "Severity",
                        "value": severity.upper(),
                        "short": True
                    },
                    {
                        "title": "Description",
                        "value": description,
                        "short": False
                    }
                ],
                "footer": "Prometheus Alert",
                "ts": int(datetime.now().timestamp())
            }]
        }

        return self.send_message(message)

if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description='Send Slack notifications')
    parser.add_argument('--webhook-url', required=True, help='Slack webhook URL')
    parser.add_argument('--type', required=True, 
                       choices=['plan', 'deployment', 'alert'],
                       help='Notification type')
    parser.add_argument('--environment', help='Environment name')
    parser.add_argument('--status', help='Status for deployment notifications')
    parser.add_argument('--message', help='Custom message')
    
    args = parser.parse_args()
    
    notifier = SlackNotifier(args.webhook_url)
    
    if args.type == 'deployment':
        success = notifier.notify_deployment_status(
            args.status, args.environment, args.message
        )
    elif args.type == 'plan':
        # Read plan output from stdin
        plan_output = sys.stdin.read()
        success = notifier.notify_terraform_plan(plan_output, args.environment)
    
    sys.exit(0 if success else 1)
```

## 🔔 Monitoring Alerts Integration

### AlertManager Slack Configuration
```yaml
# alertmanager/alertmanager.yml
global:
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
  - match:
      severity: critical
    receiver: 'critical-alerts'
  - match:
      severity: warning
    receiver: 'warning-alerts'

receivers:
- name: 'web.hook'
  slack_configs:
  - channel: '#general'
    title: 'Infrastructure Alert'
    text: 'Alert: {{ .GroupLabels.alertname }}'

- name: 'critical-alerts'
  slack_configs:
  - channel: '#alerts'
    color: 'danger'
    title: 'CRITICAL: {{ .GroupLabels.alertname }}'
    text: |
      {{ range .Alerts }}
      *Alert:* {{ .Annotations.summary }}
      *Description:* {{ .Annotations.description }}
      *Instance:* {{ .Labels.instance }}
      *Severity:* {{ .Labels.severity }}
      {{ end }}
    actions:
    - type: button
      text: 'View in Prometheus'
      url: 'http://prometheus:9090/alerts'
    - type: button
      text: 'View Grafana Dashboard'
      url: 'http://grafana:3000/d/infrastructure'

- name: 'warning-alerts'
  slack_configs:
  - channel: '#monitoring'
    color: 'warning'
    title: 'WARNING: {{ .GroupLabels.alertname }}'
    text: |
      {{ range .Alerts }}
      *Alert:* {{ .Annotations.summary }}
      *Instance:* {{ .Labels.instance }}
      {{ end }}
```

## 🤖 Devin Bot Integration

### Slack Bot Commands
```python
# slack_bot/devin_bot.py
#!/usr/bin/env python3
"""
Slack bot for Devin infrastructure management
"""

import os
import subprocess
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

app = App(token=os.environ.get("SLACK_BOT_TOKEN"))

@app.command("/devin-status")
def handle_status_command(ack, respond, command):
    """Handle /devin-status command"""
    ack()
    
    try:
        # Get infrastructure status
        result = subprocess.run(
            ['python', 'scripts/health_check.py', '--environment', 'production'],
            capture_output=True,
            text=True
        )
        
        if result.returncode == 0:
            respond({
                "response_type": "in_channel",
                "text": ":white_check_mark: Infrastructure Status: Healthy",
                "attachments": [{
                    "color": "good",
                    "text": "All systems operational"
                }]
            })
        else:
            respond({
                "response_type": "in_channel", 
                "text": ":warning: Infrastructure Status: Issues Detected",
                "attachments": [{
                    "color": "warning",
                    "text": result.stdout
                }]
            })
    except Exception as e:
        respond({
            "response_type": "ephemeral",
            "text": f":x: Error checking status: {str(e)}"
        })

@app.command("/devin-deploy")
def handle_deploy_command(ack, respond, command):
    """Handle /devin-deploy command"""
    ack()
    
    environment = command['text'].strip() or 'staging'
    
    if environment not in ['staging', 'production']:
        respond({
            "response_type": "ephemeral",
            "text": ":x: Invalid environment. Use 'staging' or 'production'"
        })
        return
    
    # Trigger deployment workflow
    respond({
        "response_type": "in_channel",
        "text": f":rocket: Deployment to {environment} initiated by <@{command['user_id']}>",
        "attachments": [{
            "color": "warning",
            "text": f"Deploying infrastructure to {environment} environment..."
        }]
    })

@app.event("app_mention")
def handle_mention(event, say):
    """Handle @devin mentions"""
    text = event['text'].lower()
    
    if 'status' in text:
        say(":mag: Checking infrastructure status...")
        # Trigger status check
    elif 'help' in text:
        say({
            "text": ":wave: Devin Infrastructure Bot Help",
            "attachments": [{
                "color": "good",
                "fields": [
                    {
                        "title": "Available Commands",
                        "value": "`/devin-status` - Check infrastructure health\n`/devin-deploy [env]` - Deploy to environment\n`@devin help` - Show this help",
                        "short": False
                    }
                ]
            }]
        })

if __name__ == "__main__":
    handler = SocketModeHandler(app, os.environ["SLACK_APP_TOKEN"])
    handler.start()
```

## 📊 Team Collaboration Features

### Daily Standup Integration
```python
# scripts/daily_standup.py
#!/usr/bin/env python3
"""
Generate daily standup reports for Slack
"""

import json
import requests
import subprocess
from datetime import datetime, timedelta

def get_recent_deployments():
    """Get deployments from last 24 hours"""
    # Query GitHub API for recent workflow runs
    # Implementation depends on your specific setup
    return []

def get_infrastructure_metrics():
    """Get key infrastructure metrics"""
    # Query Prometheus for key metrics
    # Implementation depends on your monitoring setup
    return {
        "uptime": "99.9%",
        "response_time": "150ms",
        "error_rate": "0.1%"
    }

def generate_standup_report():
    """Generate daily standup report"""
    deployments = get_recent_deployments()
    metrics = get_infrastructure_metrics()
    
    report = {
        "channel": "#daily-standup",
        "username": "Devin Daily Report",
        "icon_emoji": ":chart_with_upwards_trend:",
        "attachments": [{
            "color": "good",
            "title": f"Infrastructure Daily Report - {datetime.now().strftime('%Y-%m-%d')}",
            "fields": [
                {
                    "title": "Deployments (24h)",
                    "value": str(len(deployments)),
                    "short": True
                },
                {
                    "title": "System Uptime",
                    "value": metrics["uptime"],
                    "short": True
                },
                {
                    "title": "Avg Response Time",
                    "value": metrics["response_time"],
                    "short": True
                },
                {
                    "title": "Error Rate",
                    "value": metrics["error_rate"],
                    "short": True
                }
            ],
            "footer": "Devin Infrastructure Bot",
            "ts": int(datetime.now().timestamp())
        }]
    }
    
    return report

if __name__ == "__main__":
    webhook_url = os.environ.get("SLACK_WEBHOOK_URL")
    report = generate_standup_report()
    
    response = requests.post(webhook_url, json=report)
    print(f"Report sent: {response.status_code}")
```

## 🎯 Best Practices with Devin

### Notification Strategy
```
Slack Channel Organization:

#devops - General DevOps discussions and updates
#alerts - Critical infrastructure alerts
#deployments - Deployment notifications and status
#monitoring - System monitoring and performance updates
#incidents - Incident response and coordination
#daily-standup - Automated daily reports
```

### Common Slack Integration Requests
```
Notification Setup:
"Devin, set up Slack notifications for all infrastructure deployments"

Alert Integration:
"Devin, integrate our monitoring alerts with Slack channels"

Bot Development:
"Devin, create a Slack bot for infrastructure status and deployment commands"

Team Collaboration:
"Devin, set up automated daily reports for our DevOps team"

Incident Management:
"Devin, create Slack workflows for incident response coordination"
```

This comprehensive Slack integration ensures your team stays informed about infrastructure changes, deployments, and system health in real-time.
