# Code Review: Ansible-Based Media Stack Implementation

## Executive Summary

This is a well-structured Ansible implementation for deploying a media automation stack (Prowlarr, Sonarr, Radarr) on a home server. The code demonstrates good practices in modularity, security hardening, and maintainability. However, there are several areas for improvement around secrets management, idempotency, testing, and operational resilience.

**Overall Rating: 7.5/10**

---

## Strengths

### 1. **Excellent Role Architecture** ⭐⭐⭐⭐⭐
- The `media_app` role is a fantastic abstraction that eliminates code duplication
- Each specific app (Prowlarr, Sonarr, Radarr) uses role inclusion with parameters
- Clear separation of concerns with dedicated task files (packages, users, dirs, download, link, service)
- This design makes adding new *arr apps trivial

### 2. **Strong Security Hardening** ⭐⭐⭐⭐⭐
The systemd service template implements comprehensive hardening:
- **Filesystem isolation**: `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`
- **Capability dropping**: `CapabilityBoundingSet=` removes all capabilities
- **Syscall filtering**: Whitelist approach with `SystemCallFilter`
- **Namespace restrictions**: Prevents container escapes
- **No privilege escalation**: `NoNewPrivileges=true`
- **Read-only bind mounts** where appropriate
- This exceeds typical homelab security standards

### 3. **Smart Version Management** ⭐⭐⭐⭐
- Stores multiple versions in `/versions/` directory
- Uses symlinks for atomic rollbacks
- SHA256 checksum verification for downloads
- Blue-green deployment capability without downtime

### 4. **Automated Maintenance** ⭐⭐⭐⭐
- Self-maintaining system via systemd timer (weekly updates)
- Idempotent maintenance playbook
- Automatic reboot detection and handling
- Uses flock to prevent concurrent runs

### 5. **Code Quality Tooling** ⭐⭐⭐⭐
- Pre-commit hooks with ansible-lint
- Production profile for stricter checking
- Demonstrates commitment to code quality

---

## Issues & Recommendations

### 🔴 Critical Issues

#### 1. **No Secrets Management**
**Severity: HIGH**

**Problem:**
- No vault or secrets management solution
- API keys, passwords, and tokens would be stored in plaintext variables
- No `.gitignore` for sensitive files visible in repo

**Recommendations:**
```yaml
# Use ansible-vault for sensitive data
ansible-vault create inventories/homelab/group_vars/vault.yml

# Structure for secrets
vault_prowlarr_api_key: "xxx"
vault_sonarr_api_key: "xxx"
vault_radarr_api_key: "xxx"

# Reference in playbooks
media_app_api_key: "{{ vault_prowlarr_api_key }}"
```

**Better Alternative:**
- Use HashiCorp Vault, Bitwarden Secrets Manager, or 1Password CLI
- Implement external secrets injection via environment variables

#### 2. **Hardcoded Paths**
**Severity: MEDIUM**

**Problem:**
```ini
# ansible.cfg
inventory = /tank/appdata/ansible/inventories/homelab/hosts.ini
roles_path = /tank/appdata/ansible/roles
```

**Impact:**
- Cannot run playbooks from different locations
- Breaks portability
- Makes testing difficult

**Fix:**
```ini
[defaults]
inventory = ./inventories/homelab/hosts.ini
roles_path = ./roles:~/.ansible/roles:/usr/share/ansible/roles
```

#### 3. **Missing Backup Strategy**
**Severity: HIGH**

**Problem:**
- No backup mechanism for application data
- `/tank/appdata/*/data` contains critical configs and databases
- Single point of failure

**Recommendations:**
```yaml
# Add backup role
- name: Backup Media Apps
  hosts: control
  roles:
    - role: backup
      vars:
        backup_source: /tank/appdata
        backup_dest: /tank/backups
        backup_retention_days: 30
        backup_schedule: "daily"
```

Consider:
- Restic for encrypted, deduplicated backups
- Offsite replication (rclone to cloud storage)
- Database dumps before app updates

### 🟡 Medium Priority Issues

#### 4. **No Monitoring/Observability**
**Problem:**
- No health checks for services
- No alerting for failures
- Limited visibility into system state

**Recommendations:**
```yaml
# Add health check tasks
- name: Verify Prowlarr is responding
  ansible.builtin.uri:
    url: "http://localhost:9696/ping"
    status_code: 200
  retries: 3
  delay: 10

# Integration options
- Prometheus + Grafana for metrics
- Loki for log aggregation
- Alertmanager for notifications
- Uptime Kuma for simple status page
```

#### 5. **Incomplete Error Handling**
**Problem:**
```yaml
# download.yml - No handling for download failures
- name: Download Release
  ansible.builtin.get_url:
    url: "{{ media_app_download_url }}"
    checksum: "{{ media_app_sha256 }}"
```

**Fix:**
```yaml
- name: Download Release
  ansible.builtin.get_url:
    url: "{{ media_app_download_url }}"
    checksum: "{{ media_app_sha256 }}"
    dest: "{{ media_app_root }}/versions/{{ media_app_release }}"
  register: download_result
  retries: 3
  delay: 10
  until: download_result is succeeded
  failed_when: false

- name: Verify download succeeded
  ansible.builtin.fail:
    msg: "Failed to download {{ media_app_name }} after 3 attempts"
  when: download_result is failed
```

#### 6. **Service Dependencies Missing**
**Problem:**
- Sonarr/Radarr reference `sabnzbdplus.service` but no role exists
- References a service not defined in this codebase

**Fix:**
```yaml
# Either remove the dependency or add conditional
media_app_service_wants:
  - network-online.target
  - "{{ 'sabnzbdplus.service' if sabnzbd_enabled | default(false) else '' }}"
```

#### 7. **No Testing Framework**
**Problem:**
- No molecule tests
- No CI/CD validation
- Manual testing required

**Recommendations:**
```bash
# Add molecule for testing
molecule init scenario -r media_app

# Structure
molecule/
  default/
    molecule.yml
    converge.yml
    verify.yml
```

```yaml
# .github/workflows/ansible-test.yml
name: Ansible Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run ansible-lint
        run: ansible-lint
      - name: Run molecule tests
        run: molecule test
```

### 🟢 Low Priority Issues

#### 8. **Inconsistent Variable Naming**
**Issue in `common/tasks/main.yml`:**
```yaml
- name: Disable rsyslog
  when: journald_persist | bool  # Should be common_journald_persist
```

**Fix:**
```yaml
when: common_journald_persist | bool
```

#### 9. **Copy-Paste Error in Sonarr**
```yaml
# roles/sonarr/tasks/main.yml line 1
- name: Radarr  # Should be "Sonarr"
```

#### 10. **Missing Documentation**
**No README files for:**
- Root directory usage instructions
- Role documentation
- Playbook execution examples
- Troubleshooting guide

**Add:**
```markdown
# README.md
## Quick Start
## Prerequisites
## Installation
## Usage
## Troubleshooting

# roles/media_app/README.md
## Role Variables
## Dependencies
## Example Playbook
```

#### 11. **No Rollback Mechanism**
**Enhancement:**
```yaml
# Add rollback playbook
- name: Rollback Media App
  hosts: control
  vars_prompt:
    - name: app_name
      prompt: "Which app to rollback?"
    - name: target_version
      prompt: "Target version?"
  tasks:
    - name: Relink to previous version
      ansible.builtin.file:
        src: "/tank/appdata/{{ app_name }}/versions/{{ app_name }}_{{ target_version }}/{{ app_name | capitalize }}"
        dest: "/tank/appdata/{{ app_name }}/app"
        state: link
        force: yes
      notify: Restart app
```

#### 12. **Limited Firewall Configuration**
**Missing:**
- iptables/nftables rules
- Network segmentation
- Port restrictions

**Add:**
```yaml
# roles/common/tasks/firewall.yml
- name: Configure UFW
  community.general.ufw:
    rule: allow
    port: "{{ item }}"
    proto: tcp
  loop:
    - 9696  # Prowlarr
    - 8989  # Sonarr
    - 7878  # Radarr
```

---

## Architecture Assessment

### Directory Structure: ⭐⭐⭐⭐⭐
```
/tank/appdata/
├── prowlarr/
│   ├── data/          # SQLite databases, configs
│   ├── app/           # Symlink to active version
│   └── versions/      # All downloaded versions
├── sonarr/
└── radarr/

/srv/
├── prowlarr/          # Runtime bind mount target
├── sonarr/
└── radarr/
```

**Excellent because:**
- Clear separation of persistent data vs. application binaries
- Atomic updates via symlinks
- Easy to backup only `/data` directories
- Version history retained

**Potential Issue:**
- `/tank/appdata/ansible/` path hardcoding limits flexibility

### Service Isolation: ⭐⭐⭐⭐⭐

The systemd hardening is exceptional:
```systemd
ProtectSystem=strict
BindReadOnlyPaths={{ media_app_app }}:/srv/{{ media_app_name }}/app
BindPaths={{ media_app_data }}:/srv/{{ media_app_name }}/data
```

This creates a secure namespace where:
- Apps can't modify their own binaries
- Filesystem access is explicitly whitelisted
- Each app has minimal permissions

### User Management: ⭐⭐⭐⭐

```yaml
media_app_service_acct: arrservice  # Shared service account
media_app_owner: "{{ media_app_name }}"  # Per-app user
media_app_group: media  # Shared group
```

**Good:**
- Shared group allows inter-app file access
- Per-app users for accountability
- Proper permission inheritance (setgid bit via mode 02775)

**Consideration:**
- All apps share `arrservice` account - is this intentional?
- Document why both service account and app owner exist

---

## Security Deep Dive

### ✅ What's Secure

1. **Systemd Hardening** (lines 33-58 of app_service.j2)
   - Blocks 90% of privilege escalation vectors
   - Restricts system calls to safe subset
   - Prevents container escapes

2. **Checksum Verification**
   - SHA256 validation prevents tampering
   - Guards against MITM attacks

3. **No Root Execution**
   - All apps run as unprivileged users
   - `NoNewPrivileges=true` prevents setuid exploits

### ⚠️ Security Gaps

1. **No Network Segmentation**
   - Apps can reach any network
   - Should restrict to localhost + specific IPs
   - Add `RestrictAddressFamilies` exceptions sparingly

2. **No Rate Limiting**
   - API endpoints exposed without protection
   - Consider adding fail2ban or reverse proxy (Caddy/Traefik)

3. **Secrets in Memory**
   - No secrets management = potential for leaks
   - Implement vault integration

4. **No Intrusion Detection**
   - Consider AIDE for file integrity monitoring
   - Auditd for syscall logging

---

## Performance Considerations

### ✅ Optimizations Present

1. **Smart Fact Gathering**
   - Caching with 24h TTL reduces overhead
   - `gathering = smart` mode

2. **SSH Pipelining**
   - Reduces connection overhead
   - Good for localhost execution

3. **Efficient Updates**
   - Symlink swapping = near-zero downtime
   - Only downloads new versions when needed

### 🔧 Potential Improvements

1. **Parallel Execution**
```yaml
# Add to controller.yml
strategy: free
serial: 0  # Run all roles in parallel
```

2. **Download Caching**
```yaml
# Use local package cache
- name: Cache downloads
  ansible.builtin.copy:
    src: "{{ download_cache }}/{{ media_app_release }}"
    dest: "{{ media_app_versions }}"
  when: use_cache | default(false)
```

---

## Operational Recommendations

### 1. **Add Pre-Flight Checks**
```yaml
# playbooks/controller.yml - add pre_tasks
pre_tasks:
  - name: Verify /tank is mounted
    ansible.builtin.stat:
      path: /tank/appdata
    register: tank_mount
    failed_when: not tank_mount.stat.exists

  - name: Check disk space
    ansible.builtin.shell: df -h /tank | awk 'NR==2 {print $5}' | sed 's/%//'
    register: disk_usage
    failed_when: disk_usage.stdout | int > 90
```

### 2. **Add Service Validation**
```yaml
# roles/media_app/tasks/verify.yml
- name: Wait for service to start
  ansible.builtin.wait_for:
    port: "{{ media_app_port }}"
    timeout: 60

- name: Verify health endpoint
  ansible.builtin.uri:
    url: "http://localhost:{{ media_app_port }}/ping"
    status_code: 200
  retries: 5
  delay: 10
```

### 3. **Implement Graceful Degradation**
```yaml
# Allow playbook to continue if one app fails
- name: Deploy apps with error handling
  block:
    - include_role:
        name: "{{ item }}"
  rescue:
    - debug:
        msg: "{{ item }} failed, continuing with others"
  loop:
    - prowlarr
    - sonarr
    - radarr
```

### 4. **Add Logging Aggregation**
```yaml
# roles/logging/tasks/main.yml
- name: Configure promtail for log forwarding
  ansible.builtin.template:
    src: promtail-config.yml.j2
    dest: /etc/promtail/config.yml
```

---

## Maintenance Playbook Analysis

### ✅ Good Practices

1. **Reboot Detection**
   - Checks for `/var/run/reboot-required`
   - Automated but safe

2. **Self-Scheduling**
   - Creates its own systemd timer
   - Bootstrap once, runs forever

3. **Concurrency Protection**
   - Uses flock to prevent overlapping runs

### ⚠️ Improvements Needed

1. **No Pre-Update Backup**
```yaml
pre_tasks:
  - name: Backup application data
    ansible.builtin.command:
      cmd: restic backup /tank/appdata
```

2. **No Notification**
```yaml
post_tasks:
  - name: Send completion notification
    community.general.mail:
      subject: "Maintenance completed on {{ inventory_hostname }}"
```

3. **No Failure Handling**
```yaml
- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: dist
  register: upgrade_result
  ignore_errors: yes

- name: Alert on upgrade failure
  # Send alert
  when: upgrade_result is failed
```

---

## Code Quality Metrics

| Aspect | Rating | Notes |
|--------|--------|-------|
| Modularity | 9/10 | Excellent role abstraction |
| Idempotency | 7/10 | Mostly idempotent, some improvements needed |
| Documentation | 4/10 | Minimal inline comments, no READMEs |
| Testing | 2/10 | Only ansible-lint, no functional tests |
| Security | 9/10 | Exceptional hardening, needs secrets mgmt |
| Maintainability | 8/10 | Clean structure, easy to extend |
| Error Handling | 5/10 | Basic, needs more robustness |
| Observability | 3/10 | Logs to journald, but no monitoring |

---

## Comparison to Industry Standards

### vs. Docker Compose Approach
**Ansible Advantages:**
- Better security (systemd hardening vs. Docker's default)
- No container overhead
- Direct integration with system services
- Easier to debug (no container layers)

**Docker Advantages:**
- Faster deployment
- Better isolation
- Easier rollbacks
- Community container images

### vs. Kubernetes
**Why Ansible is Better Here:**
- Simpler for single-node deployment
- Lower resource overhead
- No orchestration complexity
- Fits homelab scale perfectly

---

## Quick Wins (Implement First)

1. **Add secrets management** (2 hours)
   ```bash
   ansible-vault create inventories/homelab/group_vars/vault.yml
   ```

2. **Fix hardcoded paths** (30 minutes)
   - Update `ansible.cfg` to use relative paths

3. **Add README.md** (1 hour)
   - Document setup, usage, troubleshooting

4. **Implement backup role** (3 hours)
   - Restic + rclone for offsite backups

5. **Add health checks** (2 hours)
   - Service validation after deployment

6. **Create rollback playbook** (2 hours)
   - Safety net for bad deployments

---

## Long-Term Roadmap

### Phase 1: Operational Excellence (1-2 weeks)
- [ ] Secrets management (Vault or ansible-vault)
- [ ] Backup automation
- [ ] Health checks and monitoring
- [ ] Documentation

### Phase 2: Reliability (2-3 weeks)
- [ ] Molecule testing framework
- [ ] CI/CD pipeline
- [ ] Alerting integration
- [ ] Log aggregation

### Phase 3: Advanced Features (1 month)
- [ ] Blue-green deployments
- [ ] Automated rollbacks
- [ ] Performance monitoring
- [ ] Disaster recovery procedures

---

## Final Verdict

**This is a solid, production-quality IaC implementation** that exceeds typical homelab standards. The code demonstrates deep knowledge of Ansible best practices, systemd hardening, and software engineering principles.

**Key Differentiators:**
- Exceptional security hardening
- Clean, modular architecture
- Minimal technical debt
- Well-suited for the problem domain

**Critical Path to Production-Ready:**
1. Add secrets management (HIGH PRIORITY)
2. Implement backups (HIGH PRIORITY)
3. Add monitoring (MEDIUM PRIORITY)
4. Write documentation (MEDIUM PRIORITY)

**Recommended Next Steps:**
1. Fix hardcoded paths in `ansible.cfg`
2. Set up ansible-vault for secrets
3. Add a backup role with restic
4. Create comprehensive README
5. Add health checks to deployment
6. Implement monitoring stack (Prometheus + Grafana)

---

## Additional Resources

### Recommended Tools
- **Secrets**: Bitwarden Secrets Manager, Vault by HashiCorp
- **Backups**: Restic, BorgBackup
- **Monitoring**: Prometheus + Grafana, Netdata
- **Testing**: Molecule, ansible-test
- **CI/CD**: GitHub Actions, GitLab CI

### Learning Materials
- Ansible Best Practices: https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html
- Systemd Hardening: https://www.freedesktop.org/software/systemd/man/systemd.exec.html
- Servarr Wiki: https://wiki.servarr.com/

---

## Contact & Questions

If you need clarification on any recommendations or want to discuss implementation approaches, feel free to ask. This codebase shows strong fundamentals and with a few targeted improvements, it could serve as a reference implementation for homelab automation.
