---
title: Implementation Roadmap
tags: [#roadmap, #planning, #implementation]
created: 2026-02-06
---

# Implementation Roadmap

## Overview

This roadmap outlines the recommended implementation path from current state to aspirational production-grade infrastructure, broken into manageable phases.

## Current State Assessment

### ✅ Completed
- [x] Ansible automation framework
- [x] Role-based architecture (media_app abstraction)
- [x] Systemd service hardening
- [x] ZFS filesystem structure
- [x] Version management via symlinks
- [x] ansible-lint integration
- [x] Bootstrap and maintenance playbooks

### ⚠️ Partially Implemented
- [ ] Documentation (minimal)
- [ ] Monitoring (journald only)
- [ ] Backups (ZFS snapshots manual)

### ❌ Missing
- [ ] Secrets management
- [ ] Automated testing (beyond lint)
- [ ] PostgreSQL migration
- [ ] Comprehensive monitoring
- [ ] Disaster recovery procedures
- [ ] CI/CD pipeline

## Implementation Phases

---

## Phase 1: Foundation Hardening (1-2 weeks)

**Objective:** Address critical security and operational gaps

### 1.1 Secrets Management

**Priority:** 🔴 CRITICAL

```yaml
# Week 1, Day 1-2 (4-6 hours)

Tasks:
1. Create ansible-vault password file
2. Encrypt sensitive variables
3. Update playbooks to use vault
4. Document vault usage

Implementation:
# Step 1: Create vault password file
echo "your-secure-password" > .vault_pass
chmod 600 .vault_pass

# Add to .gitignore
echo ".vault_pass" >> .gitignore

# Step 2: Create vault file
ansible-vault create inventories/homelab/group_vars/vault.yml

# Step 3: Add secrets
---
vault_db_passwords:
  prowlarr: "generated-password-1"
  sonarr: "generated-password-2"
  radarr: "generated-password-3"

# Step 4: Update ansible.cfg
[defaults]
vault_password_file = .vault_pass

# Step 5: Reference in playbooks
media_app_db_password: "{{ vault_db_passwords[media_app_name] }}"
```

**Validation:**
```bash
# Verify vault is encrypted
cat inventories/homelab/group_vars/vault.yml
# Should show encrypted content

# Test playbook runs
ansible-playbook playbooks/controller.yml --check
```

**Alternative (Recommended Long-term):**
- Implement [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/)
- Or [HashiCorp Vault](https://www.vaultproject.io/)

**Resources:**
- [[Documentation References#Ansible]]
- [Ansible Vault Guide](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

---

### 1.2 Fix Hardcoded Paths

**Priority:** 🟡 MEDIUM

```yaml
# Week 1, Day 3 (2 hours)

# Update ansible.cfg
[defaults]
# OLD:
# inventory = /tank/appdata/ansible/inventories/homelab/hosts.ini
# roles_path = /tank/appdata/ansible/roles

# NEW:
inventory = ./inventories/homelab/hosts.ini
roles_path = ./roles:~/.ansible/roles:/usr/share/ansible/roles
```

**Benefits:**
- Repository portable across systems
- Can run from any directory
- Easier testing and development

---

### 1.3 ZFS Dataset Structure

**Priority:** 🟡 MEDIUM

```yaml
# Week 1, Day 4-5 (4-6 hours)

Tasks:
1. Create ZFS datasets role
2. Implement per-app datasets
3. Configure snapshot policies
4. Test snapshot/restore workflow

Implementation:
# roles/zfs_datasets/tasks/main.yml
- name: Create app datasets
  community.general.zfs:
    name: "tank/appdata/{{ item }}/data"
    state: present
    extra_zfs_properties:
      recordsize: 128k
      compression: lz4
      atime: off
      com.sun:auto-snapshot: "true"
      com.sun:auto-snapshot:daily: "true"
      com.sun:auto-snapshot:weekly: "true"
  loop:
    - prowlarr
    - sonarr
    - radarr

# Integrate into controller.yml
roles:
  - zfs_datasets  # Add before common
  - common
  - prowlarr
  - sonarr
  - radarr
```

**Testing:**
```bash
# Verify datasets
zfs list -r tank/appdata

# Test snapshot
zfs snapshot tank/appdata/prowlarr/data@test

# Modify data
touch /tank/appdata/prowlarr/data/testfile

# Rollback
zfs rollback tank/appdata/prowlarr/data@test

# Verify file is gone
ls /tank/appdata/prowlarr/data/testfile
# Should not exist
```

---

### 1.4 Documentation

**Priority:** 🟡 MEDIUM

```markdown
# Week 2, Day 1-2 (6-8 hours)

Create:
1. README.md (root)
2. CONTRIBUTING.md
3. docs/ARCHITECTURE.md
4. docs/RUNBOOKS.md
5. Role-specific READMEs

# README.md Template
## Media Stack Ansible Automation

### Prerequisites
- Ubuntu 24.04 LTS
- ZFS pool at /tank
- Python 3.10+
- Ansible 2.16+

### Quick Start
1. Clone repository
2. Install dependencies: `ansible-galaxy install -r requirements.yml`
3. Configure inventory: `cp inventories/homelab/hosts.ini.example hosts.ini`
4. Run bootstrap: `ansible-playbook playbooks/bootstrap.yml`
5. Deploy stack: `ansible-playbook playbooks/controller.yml`

### Architecture
[Diagram]

### Maintenance
See docs/RUNBOOKS.md

### Troubleshooting
...
```

**Resources:**
- [Writing Good Documentation](https://www.writethedocs.org/guide/)
- [README Template](https://github.com/othneildrew/Best-README-Template)

---

## Phase 2: Operational Excellence (2-4 weeks)

**Objective:** Implement monitoring, backups, and reliability improvements

### 2.1 Monitoring Stack

**Priority:** 🔴 HIGH

```yaml
# Week 3-4 (10-15 hours)

Tasks:
1. Install Prometheus
2. Install Grafana
3. Configure node_exporter
4. Create dashboards
5. Set up basic alerts

Implementation:
# roles/prometheus/tasks/main.yml
- name: Install Prometheus
  ansible.builtin.package:
    name: prometheus
    state: present

- name: Configure Prometheus
  ansible.builtin.template:
    src: prometheus.yml.j2
    dest: /etc/prometheus/prometheus.yml
  notify: Restart Prometheus

# prometheus.yml.j2
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'zfs'
    static_configs:
      - targets: ['localhost:9134']

# Install node_exporter
- name: Install node_exporter
  ansible.builtin.package:
    name: prometheus-node-exporter
    state: present

# Install ZFS exporter
- name: Install ZFS exporter
  ansible.builtin.get_url:
    url: https://github.com/pdf/zfs_exporter/releases/download/v2.2.5/zfs_exporter
    dest: /usr/local/bin/zfs_exporter
    mode: '0755'
```

**Dashboards to Create:**
- System overview (CPU, RAM, disk)
- ZFS pool health and capacity
- Application status (up/down)
- Disk I/O and latency

**Resources:**
- [[Documentation References#Monitoring and Observability]]
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)

---

### 2.2 Backup Automation

**Priority:** 🔴 HIGH

```yaml
# Week 4-5 (8-12 hours)

Tasks:
1. Implement ZFS send/receive
2. Create backup schedule
3. Test restore procedures
4. Document DR process

Implementation:
# roles/zfs_backup/tasks/main.yml
- name: Create backup pool (if needed)
  community.general.zfs:
    name: backup-pool
    state: present

- name: Create backup snapshots
  community.general.zfs:
    name: "tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }}"
    state: present
  loop:
    - prowlarr
    - sonarr
    - radarr

- name: Replicate to backup pool
  ansible.builtin.shell: |
    LAST=$(zfs list -H -o name -t snapshot backup-pool/{{ item }}/data 2>/dev/null | tail -1 | cut -d@ -f2)
    
    if [ -n "$LAST" ]; then
      # Incremental
      zfs send -i tank/appdata/{{ item }}/data@${LAST} \
               tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }} | \
      zfs receive -F backup-pool/{{ item }}/data
    else
      # Full
      zfs send -R tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }} | \
      zfs receive backup-pool/{{ item }}/data
    fi
  loop:
    - prowlarr
    - sonarr
    - radarr

# Schedule via systemd timer
- name: Create backup timer
  ansible.builtin.copy:
    dest: /etc/systemd/system/zfs-backup.timer
    content: |
      [Unit]
      Description=Daily ZFS backup

      [Timer]
      OnCalendar=daily
      Persistent=true

      [Install]
      WantedBy=timers.target
```

**Testing Restore:**
```bash
# Restore single app from backup
zfs send backup-pool/prowlarr/data@backup-2024-01-01 | \
zfs receive tank/appdata/prowlarr/data-restored

# Verify data
ls /tank/appdata/prowlarr/data-restored/
```

---

### 2.3 Health Checks

**Priority:** 🟡 MEDIUM

```yaml
# Week 5 (4-6 hours)

Implementation:
# roles/media_app/tasks/verify.yml
- name: Wait for service to start
  ansible.builtin.wait_for:
    port: "{{ media_app_port }}"
    timeout: 60

- name: Verify API responds
  ansible.builtin.uri:
    url: "http://localhost:{{ media_app_port }}/ping"
    status_code: 200
    timeout: 10
  retries: 5
  delay: 10
  register: health_check

- name: Report health status
  ansible.builtin.debug:
    msg: "{{ media_app_name }} is {{ 'healthy' if health_check is succeeded else 'unhealthy' }}"

# Integrate into service.yml
- name: Service
  ansible.builtin.import_tasks: service.yml

- name: Verify
  ansible.builtin.import_tasks: verify.yml
  tags: [verify]
```

---

## Phase 3: Database Migration (3-6 weeks)

**Objective:** Migrate from SQLite to PostgreSQL

### 3.1 PostgreSQL Installation

**Priority:** 🟡 MEDIUM

```yaml
# Week 6-7 (10-15 hours)

Tasks:
1. Create postgres role
2. Set up ZFS dataset
3. Install PostgreSQL 16
4. Configure for workload
5. Create databases and users

Implementation:
# roles/postgres/tasks/main.yml
(See [[Database Architecture Evolution]] for complete implementation)

Key steps:
1. ZFS dataset with recordsize=8k
2. PostgreSQL 16 installation
3. Configuration tuning
4. Database creation
5. User/permission setup
```

**Resources:**
- [[Database Architecture Evolution]]
- [[Documentation References#PostgreSQL]]

---

### 3.2 Migration Execution

**Priority:** 🟡 MEDIUM

```yaml
# Week 8-9 (15-20 hours)

Tasks:
1. Test migration on one app (Prowlarr)
2. Validate data integrity
3. Performance testing
4. Migrate remaining apps
5. Update backup strategy

Migration per app:
1. Create pre-migration snapshot
2. Stop application
3. Update config.xml for PostgreSQL
4. Start application (auto-migrates)
5. Verify migration
6. Monitor for 24-48 hours
7. Repeat for next app
```

**Rollback Plan:**
```bash
# If migration fails
systemctl stop prowlarr
zfs rollback tank/appdata/prowlarr/data@pre-postgres
# Revert config.xml to SQLite
systemctl start prowlarr
```

---

### 3.3 PostgreSQL Backups

**Priority:** 🔴 HIGH

```yaml
# Week 9-10 (6-8 hours)

Tasks:
1. Implement pg_dump backups
2. Configure WAL archiving
3. Test PITR (Point-in-Time Recovery)
4. Document procedures

Implementation:
# Daily backups
- name: Backup PostgreSQL databases
  ansible.builtin.shell: |
    pg_dump -Fc {{ item }}_db > /backup/postgres/{{ item }}_{{ ansible_date_time.date }}.dump
  loop:
    - prowlarr
    - sonarr
    - radarr
  become_user: postgres

# WAL archiving (optional, for PITR)
# postgresql.conf
archive_mode = on
archive_command = 'cp %p /backup/postgres/wal/%f'
```

---

## Phase 4: Advanced Features (2-3 months)

**Objective:** Production-grade automation and testing

### 4.1 Automated Testing

**Priority:** 🟡 MEDIUM

```yaml
# Weeks 10-12 (15-20 hours)

Tasks:
1. Install Molecule
2. Create test scenarios
3. Implement CI/CD
4. Add integration tests

Implementation:
# Install molecule
pip install molecule molecule-plugins[docker]

# Initialize test scenario
cd roles/media_app
molecule init scenario -r media_app

# molecule/default/converge.yml
- name: Converge
  hosts: all
  roles:
    - role: media_app
      vars:
        media_app_name: testapp
        media_app_version: 1.0.0

# molecule/default/verify.yml
- name: Verify
  hosts: all
  tasks:
    - name: Check service is running
      ansible.builtin.systemd:
        name: testapp.service
        state: started
      register: service_state
      failed_when: service_state.status.ActiveState != "active"
```

**Resources:**
- [[Documentation References#Ansible]]
- [Molecule Documentation](https://molecule.readthedocs.io/)

---

### 4.2 CI/CD Pipeline

**Priority:** 🟡 MEDIUM

```yaml
# Weeks 12-14 (10-15 hours)

# .github/workflows/ansible-test.yml
name: Ansible Tests

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run ansible-lint
        run: |
          pip install ansible-lint
          ansible-lint

  molecule:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run molecule tests
        run: |
          pip install molecule molecule-plugins[docker]
          cd roles/media_app
          molecule test

  deploy-staging:
    needs: [lint, molecule]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to staging
        run: ansible-playbook -i staging playbooks/controller.yml
```

---

### 4.3 Disaster Recovery

**Priority:** 🟡 MEDIUM

```yaml
# Weeks 14-16 (10-12 hours)

Tasks:
1. Document DR procedures
2. Create DR playbook
3. Test full restore
4. Automate DR testing

# playbooks/disaster_recovery.yml
- name: Disaster Recovery
  hosts: control
  tasks:
    - name: Stop all services
      # ...
    
    - name: Restore from backups
      # ZFS send/receive from backup pool
    
    - name: Restore PostgreSQL
      # pg_restore from dumps
    
    - name: Start services
      # ...
    
    - name: Verify functionality
      # Health checks
```

**DR Testing Schedule:**
- Quarterly full restore test
- Monthly snapshot restore test
- Document results and timing

---

## Timeline Summary

```
Phase 1: Foundation (Weeks 1-2)
├─ Secrets management
├─ Fix hardcoded paths
├─ ZFS datasets
└─ Documentation

Phase 2: Operations (Weeks 3-5)
├─ Monitoring (Prometheus/Grafana)
├─ Backup automation
└─ Health checks

Phase 3: Database (Weeks 6-10)
├─ PostgreSQL setup
├─ Migration execution
└─ PostgreSQL backups

Phase 4: Advanced (Weeks 10-16)
├─ Testing framework
├─ CI/CD pipeline
└─ Disaster recovery

Total: ~4 months part-time
```

## Success Criteria

### Phase 1
- [ ] No plaintext secrets in repository
- [ ] Can run playbooks from any directory
- [ ] ZFS snapshots automated
- [ ] Comprehensive README exists

### Phase 2
- [ ] Grafana dashboards showing metrics
- [ ] Automated daily backups
- [ ] Health checks pass after deployment
- [ ] Alert on service failures

### Phase 3
- [ ] All apps on PostgreSQL
- [ ] pg_dump backups working
- [ ] Rollback tested successfully
- [ ] Performance acceptable

### Phase 4
- [ ] Molecule tests passing
- [ ] CI/CD pipeline green
- [ ] DR tested within RTO
- [ ] Full documentation complete

## Risk Mitigation

### High-Risk Activities

**Database Migration:**
- **Risk:** Data loss or corruption
- **Mitigation:** ZFS snapshots before migration, extensive testing, gradual rollout

**Backup Changes:**
- **Risk:** Untested backups fail when needed
- **Mitigation:** Regular restore testing, verify backup integrity

**Production Changes:**
- **Risk:** Service downtime
- **Mitigation:** Maintenance windows, rollback procedures, health checks

## Resources Required

### Time Investment
- **Phase 1:** 16-24 hours
- **Phase 2:** 24-35 hours
- **Phase 3:** 35-50 hours
- **Phase 4:** 35-50 hours
- **Total:** ~110-160 hours (3-4 months part-time)

### Financial
- **Hardware:** None (existing server sufficient)
- **Software:** Free (open source tools)
- **Training:** $0-800 (optional certifications)
- **Total:** $0-800

### Learning
- Ansible: Advanced
- PostgreSQL: Intermediate
- Monitoring: Intermediate
- Testing: Beginner to Intermediate

## Related Notes

- [[Technical Architecture Overview]]
- [[Required Skills Matrix]]
- [[Industry Standard Comparisons]]
- [[Database Architecture Evolution]]
- [[Documentation References]]

---

**Next Actions:**
1. Review roadmap and adjust timeline to your availability
2. Start with Phase 1, Week 1: Secrets management
3. Document progress and learnings
4. Iterate based on experience
