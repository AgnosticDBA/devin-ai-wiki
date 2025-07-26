# Ansible Playbooks with Devin

Enhance your Ansible automation with Devin's expertise in configuration management, role development, and playbook optimization. This guide provides practical examples for leveraging Devin in your Ansible workflows.

## 🎯 Core Ansible Tasks with Devin

### Playbook Analysis and Optimization
Devin can analyze your existing playbooks and suggest improvements:

```
Example Request:
"Can you review our playbook.yml and suggest optimizations for better 
performance and maintainability?"

Devin's Analysis of Current Playbook:
- 87 lines with extensive role list
- Potential for role consolidation
- Missing error handling
- No environment-specific configurations
- Opportunities for parallel execution
```

### Role Development and Enhancement
```yaml
# Example: Devin-enhanced web server role
# roles/web/apache2/tasks/main.yml
---
- name: Update package cache
  apt:
    update_cache: yes
    cache_valid_time: 3600
  tags: ['packages']

- name: Install Apache2 and required modules
  apt:
    name:
      - apache2
      - apache2-utils
      - libapache2-mod-ssl
      - libapache2-mod-security2
    state: present
  notify: restart apache2
  tags: ['packages', 'apache']

- name: Enable Apache2 modules
  apache2_module:
    name: "{{ item }}"
    state: present
  loop:
    - rewrite
    - ssl
    - headers
    - security2
  notify: restart apache2
  tags: ['apache', 'modules']

- name: Configure Apache2 security settings
  template:
    src: security.conf.j2
    dest: /etc/apache2/conf-available/security.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: restart apache2
  tags: ['apache', 'security']

- name: Enable security configuration
  command: a2enconf security
  args:
    creates: /etc/apache2/conf-enabled/security.conf
  notify: restart apache2
  tags: ['apache', 'security']

- name: Create virtual host configuration
  template:
    src: vhost.conf.j2
    dest: "/etc/apache2/sites-available/{{ apache_vhost_name }}.conf"
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: restart apache2
  tags: ['apache', 'vhost']

- name: Enable virtual host
  command: "a2ensite {{ apache_vhost_name }}"
  args:
    creates: "/etc/apache2/sites-enabled/{{ apache_vhost_name }}.conf"
  notify: restart apache2
  tags: ['apache', 'vhost']

- name: Disable default site
  command: a2dissite 000-default
  args:
    removes: /etc/apache2/sites-enabled/000-default.conf
  notify: restart apache2
  tags: ['apache', 'vhost']

- name: Start and enable Apache2 service
  systemd:
    name: apache2
    state: started
    enabled: yes
    daemon_reload: yes
  tags: ['apache', 'service']

- name: Verify Apache2 is responding
  uri:
    url: "http://{{ ansible_default_ipv4.address }}"
    method: GET
    status_code: 200
  retries: 3
  delay: 5
  tags: ['apache', 'verification']
```

## 🚀 Advanced Ansible Patterns

### Dynamic Inventory Management
```python
#!/usr/bin/env python3
# inventory/openstack_inventory.py - Devin-generated dynamic inventory

import json
import openstack
import sys

def get_openstack_inventory():
    """Generate dynamic inventory from OpenStack"""
    conn = openstack.connect()
    
    inventory = {
        '_meta': {
            'hostvars': {}
        },
        'all': {
            'children': ['web_servers', 'database_servers', 'monitoring_servers']
        },
        'web_servers': {
            'hosts': []
        },
        'database_servers': {
            'hosts': []
        },
        'monitoring_servers': {
            'hosts': []
        }
    }
    
    # Get all servers
    servers = conn.compute.servers()
    
    for server in servers:
        if server.status == 'ACTIVE':
            hostname = server.name
            ip_address = server.access_ipv4 or server.access_ipv6
            
            # Add to appropriate group based on server name
            if 'web' in hostname:
                inventory['web_servers']['hosts'].append(hostname)
            elif 'db' in hostname:
                inventory['database_servers']['hosts'].append(hostname)
            elif 'monitoring' in hostname:
                inventory['monitoring_servers']['hosts'].append(hostname)
            
            # Add host variables
            inventory['_meta']['hostvars'][hostname] = {
                'ansible_host': ip_address,
                'openstack_id': server.id,
                'openstack_flavor': server.flavor['original_name'],
                'openstack_image': server.image['name'],
                'openstack_az': server.availability_zone,
                'ansible_user': 'ubuntu',
                'ansible_ssh_private_key_file': '~/.ssh/openstack_key'
            }
    
    return inventory

if __name__ == '__main__':
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        print(json.dumps(get_openstack_inventory(), indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        print(json.dumps({}))
    else:
        print("Usage: %s --list or %s --host <hostname>" % (sys.argv[0], sys.argv[0]))
```

### Environment-Specific Playbooks
```yaml
# playbooks/site.yml - Main site playbook
---
- import_playbook: common.yml
- import_playbook: web_servers.yml
- import_playbook: database_servers.yml
- import_playbook: monitoring.yml

# playbooks/web_servers.yml
---
- name: Configure web servers
  hosts: web_servers
  become: yes
  vars_files:
    - "../group_vars/{{ environment }}/web.yml"
    - "../group_vars/{{ environment }}/vault.yml"
  
  pre_tasks:
    - name: Wait for system to become reachable
      wait_for_connection:
        timeout: 300
      tags: ['always']
    
    - name: Gather facts
      setup:
      tags: ['always']
  
  roles:
    - role: common
      tags: ['common']
    - role: security
      tags: ['security']
    - role: web/apache2
      tags: ['web', 'apache']
    - role: monitoring/node_exporter
      tags: ['monitoring']
  
  post_tasks:
    - name: Verify web service is accessible
      uri:
        url: "http://{{ ansible_default_ipv4.address }}"
        method: GET
        status_code: 200
      delegate_to: localhost
      tags: ['verification']

# playbooks/database_servers.yml
---
- name: Configure database servers
  hosts: database_servers
  become: yes
  vars_files:
    - "../group_vars/{{ environment }}/database.yml"
    - "../group_vars/{{ environment }}/vault.yml"
  
  roles:
    - role: common
      tags: ['common']
    - role: security
      tags: ['security']
    - role: database/mysql
      tags: ['database', 'mysql']
    - role: backup
      tags: ['backup']
    - role: monitoring/mysql_exporter
      tags: ['monitoring']
```

### Advanced Role Structure
```yaml
# roles/security/tasks/main.yml - Comprehensive security role
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"
  tags: ['always']

- name: Include environment-specific tasks
  include_tasks: "{{ environment }}.yml"
  when: environment is defined
  tags: ['always']

- name: Configure SSH hardening
  include_tasks: ssh_hardening.yml
  tags: ['ssh', 'hardening']

- name: Configure firewall
  include_tasks: firewall.yml
  tags: ['firewall', 'security']

- name: Install and configure fail2ban
  include_tasks: fail2ban.yml
  tags: ['fail2ban', 'security']

- name: Configure automatic updates
  include_tasks: auto_updates.yml
  tags: ['updates', 'security']

- name: Harden system settings
  include_tasks: system_hardening.yml
  tags: ['hardening', 'security']

# roles/security/tasks/ssh_hardening.yml
---
- name: Backup original SSH configuration
  copy:
    src: /etc/ssh/sshd_config
    dest: /etc/ssh/sshd_config.backup
    remote_src: yes
    owner: root
    group: root
    mode: '0600'
  tags: ['ssh', 'backup']

- name: Configure SSH daemon
  template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: '0600'
    backup: yes
    validate: '/usr/sbin/sshd -t -f %s'
  notify: restart ssh
  tags: ['ssh', 'config']

- name: Generate new SSH host keys
  command: "ssh-keygen -t {{ item.type }} -b {{ item.bits }} -f /etc/ssh/ssh_host_{{ item.type }}_key -N ''"
  args:
    creates: "/etc/ssh/ssh_host_{{ item.type }}_key"
  loop:
    - { type: 'rsa', bits: 4096 }
    - { type: 'ed25519', bits: 256 }
  notify: restart ssh
  tags: ['ssh', 'keys']

- name: Set proper permissions on SSH host keys
  file:
    path: "/etc/ssh/ssh_host_{{ item.type }}_key"
    owner: root
    group: root
    mode: '0600'
  loop:
    - { type: 'rsa' }
    - { type: 'ed25519' }
  tags: ['ssh', 'permissions']

- name: Remove weak SSH host keys
  file:
    path: "/etc/ssh/ssh_host_{{ item }}_key"
    state: absent
  loop:
    - dsa
    - ecdsa
  notify: restart ssh
  tags: ['ssh', 'cleanup']
```

## 🔧 Ansible Workflow Automation

### Testing and Validation
```yaml
# molecule/default/molecule.yml - Testing configuration
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-20.04
    image: ubuntu:20.04
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
provisioner:
  name: ansible
  config_options:
    defaults:
      interpreter_python: auto_silent
      callback_whitelist: profile_tasks, timer, yaml
    ssh_connection:
      pipelining: false
verifier:
  name: ansible
```

### CI/CD Integration
```yaml
# .github/workflows/ansible-ci.yml
name: Ansible CI

on:
  push:
    paths: ['ansible/**', 'playbooks/**', 'roles/**']
  pull_request:
    paths: ['ansible/**', 'playbooks/**', 'roles/**']

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          pip install ansible ansible-lint yamllint
      
      - name: Run YAML Lint
        run: yamllint .
      
      - name: Run Ansible Lint
        run: ansible-lint playbooks/ roles/
      
      - name: Syntax check
        run: |
          ansible-playbook --syntax-check playbooks/site.yml

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-20.04, ubuntu-22.04]
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Molecule tests
        run: |
          pip install molecule[docker]
          molecule test
```

## 📊 Monitoring and Observability

### Ansible-Managed Monitoring
```yaml
# roles/monitoring/prometheus/tasks/main.yml
---
- name: Create prometheus user
  user:
    name: prometheus
    system: yes
    shell: /bin/false
    home: /var/lib/prometheus
    createhome: no
  tags: ['prometheus', 'user']

- name: Create prometheus directories
  file:
    path: "{{ item }}"
    state: directory
    owner: prometheus
    group: prometheus
    mode: '0755'
  loop:
    - /etc/prometheus
    - /var/lib/prometheus
    - /var/lib/prometheus/data
  tags: ['prometheus', 'directories']

- name: Download and install Prometheus
  unarchive:
    src: "https://github.com/prometheus/prometheus/releases/download/v{{ prometheus_version }}/prometheus-{{ prometheus_version }}.linux-amd64.tar.gz"
    dest: /tmp
    remote_src: yes
    creates: "/tmp/prometheus-{{ prometheus_version }}.linux-amd64"
  tags: ['prometheus', 'install']

- name: Copy Prometheus binaries
  copy:
    src: "/tmp/prometheus-{{ prometheus_version }}.linux-amd64/{{ item }}"
    dest: "/usr/local/bin/{{ item }}"
    remote_src: yes
    owner: root
    group: root
    mode: '0755'
  loop:
    - prometheus
    - promtool
  notify: restart prometheus
  tags: ['prometheus', 'install']

- name: Configure Prometheus
  template:
    src: prometheus.yml.j2
    dest: /etc/prometheus/prometheus.yml
    owner: prometheus
    group: prometheus
    mode: '0644'
  notify: restart prometheus
  tags: ['prometheus', 'config']

- name: Create Prometheus systemd service
  template:
    src: prometheus.service.j2
    dest: /etc/systemd/system/prometheus.service
    owner: root
    group: root
    mode: '0644'
  notify:
    - reload systemd
    - restart prometheus
  tags: ['prometheus', 'service']

- name: Start and enable Prometheus
  systemd:
    name: prometheus
    state: started
    enabled: yes
    daemon_reload: yes
  tags: ['prometheus', 'service']
```

## 🛡️ Security and Compliance

### Vault Integration
```yaml
# group_vars/all/vault.yml (encrypted with ansible-vault)
$ANSIBLE_VAULT;1.1;AES256
66386439653...

# Decrypted content:
vault_mysql_root_password: "super_secure_password"
vault_api_keys:
  monitoring: "monitoring_api_key"
  backup: "backup_service_key"
vault_ssl_certificates:
  web_cert: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  web_key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----

# Usage in playbooks
- name: Configure MySQL root password
  mysql_user:
    name: root
    password: "{{ vault_mysql_root_password }}"
    login_unix_socket: /var/run/mysqld/mysqld.sock
  no_log: true
  tags: ['mysql', 'security']
```

### Compliance Automation
```yaml
# roles/compliance/cis/tasks/main.yml - CIS benchmark implementation
---
- name: CIS 1.1.1.1 - Ensure mounting of cramfs filesystems is disabled
  lineinfile:
    path: /etc/modprobe.d/blacklist-rare-network.conf
    line: "install cramfs /bin/true"
    create: yes
  tags: ['cis', 'filesystem']

- name: CIS 1.1.1.2 - Ensure mounting of freevxfs filesystems is disabled
  lineinfile:
    path: /etc/modprobe.d/blacklist-rare-network.conf
    line: "install freevxfs /bin/true"
    create: yes
  tags: ['cis', 'filesystem']

- name: CIS 1.4.1 - Ensure permissions on bootloader config are configured
  file:
    path: /boot/grub/grub.cfg
    owner: root
    group: root
    mode: '0600'
  tags: ['cis', 'bootloader']

- name: CIS 2.2.1.1 - Ensure time synchronization is in use
  package:
    name: ntp
    state: present
  tags: ['cis', 'time']

- name: CIS 3.1.1 - Ensure IP forwarding is disabled
  sysctl:
    name: net.ipv4.ip_forward
    value: '0'
    state: present
    reload: yes
  tags: ['cis', 'network']
```

## 🎯 Best Practices with Devin

### Playbook Organization
```
Ansible Project Structure (Devin-recommended):

ansible/
├── playbooks/
│   ├── site.yml
│   ├── web_servers.yml
│   ├── database_servers.yml
│   └── monitoring.yml
├── roles/
│   ├── common/
│   ├── security/
│   ├── web/
│   ├── database/
│   └── monitoring/
├── inventory/
│   ├── production/
│   ├── staging/
│   └── development/
├── group_vars/
│   ├── all/
│   ├── web_servers/
│   └── database_servers/
├── host_vars/
├── library/
├── filter_plugins/
└── ansible.cfg
```

### Common Devin Requests for Ansible
```
Role Development:
"Create a comprehensive MySQL role with security hardening and backup"

Playbook Optimization:
"Optimize our playbook for faster execution and better error handling"

Security Implementation:
"Implement CIS benchmarks across all our servers"

Monitoring Setup:
"Add comprehensive monitoring to all services managed by Ansible"

Compliance Automation:
"Create playbooks for SOC 2 compliance requirements"

Testing Integration:
"Set up Molecule testing for all our custom roles"
```

This comprehensive approach to Ansible automation with Devin ensures efficient, secure, and maintainable configuration management for your infrastructure.
