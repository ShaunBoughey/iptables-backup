# Ansible iptables Backup Role

This role provides automated backup of iptables rules from multiple hosts with Git version control, retention management, and cleanup.

## Features

- **Multi-Host Backup**: Backs up iptables rules from all hosts in the `gateways` group
- **Git Integration**: Optional Git repository for version control and change tracking
- **Retention Management**: Automatic cleanup of old backup files based on configurable retention period
- **Timestamped Backups**: Each backup is timestamped for easy identification

## Quick Start

### 1. Inventory Configuration
```ini
[gateways]
gateway1 ansible_host=192.168.1.1 ansible_port=22
gateway2 ansible_host=192.168.1.2 ansible_port=22
```

### 2. Playbook

```yaml
---
- name: Backup iptables
  hosts: gateways
  gather_facts: yes
  
  pre_tasks:
    - name: Set timestamp
      set_fact:
        timestamp: "{{ ansible_date_time.year }}{{ ansible_date_time.month }}{{ ansible_date_time.day }}_{{ ansible_date_time.hour }}{{ ansible_date_time.minute }}{{ ansible_date_time.second }}"

  roles:
    - iptables_backup
```

### 3. Deploy

```bash
ansible-playbook -i inventory.yml backup-iptables.yml --ask-become-pass
```

## Backup Structure

The role creates the following directory structure:

```
backups/
├── .git/                    # Git repository (if enabled)
├── gateway1/
│   ├── 20241201_143022.iptables
│   ├── 20241202_143015.iptables
│   └── ...
└── gateway2/
    ├── 20241201_143025.iptables
    ├── 20241202_143018.iptables
    └── ...
```

## Variable Reference

### Default Variables
- `backup_dir`: Directory to store backups (default: `{{ playbook_dir }}/backups`)
- `retention_days`: Number of days to keep backup files (default: `30`)
- `git_user`: Git username for commits (default: `"Ansible"`)
- `git_email`: Git email for commits (default: `"ansible@localhost"`)
- `git_enabled`: Enable/disable Git integration (default: `true`)

### Override Example
```yaml
# In your playbook or inventory
vars:
  backup_dir: "/opt/iptables-backups"
  retention_days: 14
  git_user: "Admin"
  git_email: "admin@company.com"
  git_enabled: false
```

## Git Integration

When `git_enabled` is `true`, the role will:
- Initialize a Git repository in the backup directory (if it doesn't exist)
- Configure Git user settings
- Commit new backups with descriptive messages
- Commit cleanup operations when old files are removed

### Git Commit Messages
- New backups: `"Backup YYYYMMDD_HHMMSS - gateway1, gateway2"`
- Cleanup operations: `"Cleanup old backups (older than 30 days)"`

## Monitoring Backup Status

### Check Recent Backups
```bash
ls -la backups/*/
```

### Verify Backup Content
```bash
# View latest backup for a specific host
cat backups/gateway1/$(ls -t backups/gateway1/ | head -1)
```

## Restoring IPTables Rules

To restore iptables rules from a backup:

```bash
# On the target host
sudo iptables-restore < /path/to/backup.iptables

# Or via Ansible
ansible gateway1 -i inventory.yml -m shell -a "iptables-restore < /path/to/backup.iptables" --become
```

## Automation

### Cron Job Example
```bash
# Add to crontab for daily backups at 2 AM
0 2 * * * cd /path/to/ansible && ansible-playbook -i inventory.yml backup-iptables.yml
```