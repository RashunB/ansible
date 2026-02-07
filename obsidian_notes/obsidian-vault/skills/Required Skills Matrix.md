---
title: Required Skills Matrix
tags: [#skills, #learning, #career-development]
created: 2026-02-06
---

# Required Skills Matrix

## Skill Assessment Framework

### Skill Levels Defined

| Level | Description | Indicators |
|-------|-------------|------------|
| **Novice** | Basic awareness, requires guidance | Can follow tutorials, limited troubleshooting |
| **Intermediate** | Working knowledge, can operate independently | Can solve common problems, understands concepts |
| **Advanced** | Deep understanding, can design solutions | Can optimize, debug complex issues, teach others |
| **Expert** | Mastery, recognized authority | Contributes to projects, writes documentation, speaks at conferences |

## Skills Required for This Implementation

### Core Infrastructure Skills

#### Linux System Administration
**Required Level:** Intermediate to Advanced

**Specific Competencies:**
- [ ] File system hierarchy (FHS) understanding
- [ ] Process management (ps, top, systemctl)
- [ ] User and permission management (chmod, chown, ACLs)
- [ ] Package management (apt, dnf, yum)
- [ ] System logging (journald, syslog, logrotate)
- [ ] Network configuration (systemd-networkd, netplan)
- [ ] SSH key management and configuration

**Learning Resources:**
- [Linux Journey](https://linuxjourney.com/) - Interactive learning path
- [The Linux Command Line (William Shotts)](https://linuxcommand.org/tlcl.php) - Free book
- [Red Hat System Administration I (RH124)](https://www.redhat.com/en/services/training/rh124-red-hat-system-administration-i)
- [Linux Foundation Certified SysAdmin](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/)

**Testing Your Level:**
```bash
# Can you explain what these do and when to use them?
sudo systemctl status sshd.service
sudo journalctl -u sshd -f --since "1 hour ago"
sudo lsof -i :22
sudo netstat -tulpn | grep :22
```

---

#### Systemd Service Management
**Required Level:** Advanced

**Specific Competencies:**
- [ ] Unit file creation and templating
- [ ] Service dependencies (Wants, Requires, After, Before)
- [ ] Security hardening directives (ProtectSystem, CapabilityBoundingSet, etc.)
- [ ] Namespace isolation (PrivateTmp, PrivateDevices, etc.)
- [ ] Resource limits (CPUQuota, MemoryMax, TasksMax)
- [ ] Timers and automation

**Learning Resources:**
- [systemd Documentation](https://www.freedesktop.org/software/systemd/man/)
- [Understanding Systemd Units (DigitalOcean)](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files)
- [Systemd for Administrators Series](https://www.freedesktop.org/wiki/Software/systemd/TipsAndTricks/)
- [The systemd-less-travelled (Lennart Poettering Blog)](https://0pointer.de/blog/projects/)

**Testing Your Level:**
```bash
# Can you create a hardened service from scratch?
# Can you debug why a service won't start?
# Can you explain the difference between Type=simple and Type=forking?

# Advanced challenge:
systemd-analyze security prowlarr.service
# Can you interpret and improve the security score?
```

**Implementation Examples from This Project:**
```systemd
# From app_service.j2
NoNewPrivileges=true           # Prevents privilege escalation
ProtectSystem=strict           # Read-only /usr, /boot, /etc
CapabilityBoundingSet=         # Drops ALL Linux capabilities
SystemCallFilter=@system-service # Whitelist syscalls only
```

---

#### ZFS Administration
**Required Level:** Intermediate to Advanced

**Specific Competencies:**
- [ ] Pool creation and management (zpool create, add, destroy)
- [ ] Dataset hierarchy design
- [ ] Snapshot and clone operations
- [ ] Send/receive for backups
- [ ] Property tuning (recordsize, compression, atime)
- [ ] Scrub and resilver understanding
- [ ] ARC and L2ARC tuning
- [ ] Monitoring and health checks

**Learning Resources:**
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)
- [FreeBSD Handbook: ZFS Chapter](https://docs.freebsd.org/en/books/handbook/zfs/)
- [FreeBSD Mastery: ZFS (Michael W Lucas)](https://www.tiltedwindmillpress.com/product/fmzfs/)
- [Oracle ZFS Administration Guide](https://docs.oracle.com/cd/E19253-01/819-5461/zfsover-2/)
- [JRS Systems: ZFS Best Practices](https://jrs-s.net/category/open-source/zfs/)

**Testing Your Level:**
```bash
# Can you answer these?
# - When to use recordsize=128k vs 8k vs 1M?
# - What's the difference between snapshot and clone?
# - How much RAM do you need for deduplication?
# - How to recover from a degraded pool?

# Practical test:
zfs list -o space -r tank/appdata
zfs get all tank/appdata/prowlarr/data | grep -E 'recordsize|compression|atime'
```

**Key Concepts Tested:**
```bash
# Copy-on-write implications
zfs snapshot tank/appdata/test@before
# ... make changes ...
zfs list -t snapshot tank/appdata/test
zfs diff tank/appdata/test@before
zfs rollback tank/appdata/test@before
```

---

### Automation and Configuration Management

#### Ansible
**Required Level:** Advanced

**Specific Competencies:**
- [ ] Playbook design and structure
- [ ] Role development (tasks, handlers, defaults, templates)
- [ ] Variable precedence and scoping
- [ ] Jinja2 templating
- [ ] Inventory management (static and dynamic)
- [ ] Vault for secrets management
- [ ] Module development (when needed)
- [ ] Idempotency patterns
- [ ] Error handling and rescue blocks
- [ ] Performance optimization (fact caching, pipelining)

**Learning Resources:**
- [Ansible Official Documentation](https://docs.ansible.com/)
- [Ansible for DevOps (Jeff Geerling)](https://www.ansiblefordevops.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Learn Ansible (Free Video Series)](https://www.youtube.com/playlist?list=PL2_OBreMn7FqZkvMYt6ATmgC0KAGGJNAN)
- [Red Hat Ansible Automation Platform Specialist](https://www.redhat.com/en/services/training/ex374-red-hat-certified-specialist-ansible-automation-exam)

**Testing Your Level:**
```yaml
# Can you explain why this is idempotent?
- name: Create symbolic link
  ansible.builtin.file:
    src: "{{ source }}"
    dest: "{{ dest }}"
    state: link
    force: true

# Can you refactor this for better reusability?
# (Hint: The media_app role is the answer)
```

**Advanced Pattern Recognition:**
```yaml
# This project demonstrates:
# 1. Template Method pattern (media_app role)
# 2. DRY principle (role inclusion)
# 3. Separation of concerns (task files)
# 4. Handler usage for efficiency
# 5. Variable layering (defaults -> group_vars -> host_vars)
```

---

#### YAML and Jinja2
**Required Level:** Intermediate

**Specific Competencies:**
- [ ] YAML syntax (lists, dictionaries, multiline strings)
- [ ] Jinja2 filters and tests
- [ ] Template logic (loops, conditionals)
- [ ] Variable interpolation
- [ ] Whitespace control

**Learning Resources:**
- [YAML Specification](https://yaml.org/spec/)
- [Jinja2 Documentation](https://jinja.palletsprojects.com/)
- [Ansible Filters](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html)

**Testing Your Level:**
```jinja
{# Can you predict the output? #}
{% for path in media_app_service_bind_ro | default([]) %}
BindReadOnlyPaths={{ path }}
{% endfor %}

{# When would this produce empty output? #}
{# How would you modify it to add a default path? #}
```

---

### Security and Hardening

#### Linux Security Concepts
**Required Level:** Advanced

**Specific Competencies:**
- [ ] Principle of least privilege
- [ ] Linux capabilities vs full root
- [ ] Namespaces and cgroups
- [ ] Syscall filtering (seccomp)
- [ ] SELinux/AppArmor basics
- [ ] Filesystem permissions and ACLs
- [ ] Network security (iptables/nftables)

**Learning Resources:**
- [Linux Kernel Security Subsystems](https://www.kernel.org/doc/html/latest/security/index.html)
- [NIST Security Guidelines](https://www.nist.gov/programs-projects/security-content-automation-protocol-scap)
- [CIS Benchmarks (Linux)](https://www.cisecurity.org/cis-benchmarks/)
- [Linux Security Summit Presentations](https://events.linuxfoundation.org/linux-security-summit-north-america/)

**Understanding This Project's Hardening:**

```systemd
# No new privileges (prevents suid exploits)
NoNewPrivileges=true

# Drops all Linux capabilities
# (app can't bind low ports, modify kernel params, etc.)
CapabilityBoundingSet=

# Syscall whitelist (blocks dangerous operations)
SystemCallFilter=@system-service @network-io
SystemCallFilter=~ @mount @reboot @swap

# Filesystem protection
ProtectSystem=strict      # /usr, /boot, /etc read-only
ProtectHome=true          # No access to /home
PrivateTmp=true          # Isolated /tmp
ProtectKernelTunables=true # Can't modify /proc/sys

# Network restrictions
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
```

**Security Score Comparison:**
```bash
# Run on your implementation
systemd-analyze security prowlarr.service
# Overall exposure level: 1.7 (SAFE)

# Typical Docker container (no hardening)
# Overall exposure level: 9.2 (UNSAFE)
```

**Reference:**
- [systemd Security Features](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Security)
- [Understanding Linux Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)

---

### Database Management

#### SQLite
**Required Level:** Intermediate

**Specific Competencies:**
- [ ] Database file structure
- [ ] Backup strategies (SQLite is just a file)
- [ ] VACUUM and optimization
- [ ] PRAGMA settings
- [ ] WAL (Write-Ahead Logging) mode
- [ ] Concurrent access patterns

**Learning Resources:**
- [SQLite Official Documentation](https://www.sqlite.org/docs.html)
- [SQLite for Application Developers](https://www.sqlite.org/appfileformat.html)
- [Use The Index, Luke (SQL Performance)](https://use-the-index-luke.com/)

**Why SQLite for This Stack:**
- Single-user applications (Sonarr, Radarr, Prowlarr)
- Low write volume (metadata, not media files)
- Simple backup (just copy the file)
- Zero administration overhead

**Limitations to Know:**
- No network access (local file only)
- Limited concurrency (one writer at a time)
- No user/permission system
- Not ideal for high-write workloads

---

#### PostgreSQL (Future Target)
**Required Level:** Intermediate

**Specific Competencies:**
- [ ] Installation and configuration
- [ ] Database and user management
- [ ] Connection pooling (pgBouncer)
- [ ] Backup strategies (pg_dump, pg_basebackup)
- [ ] Performance tuning (shared_buffers, work_mem)
- [ ] Monitoring and logging
- [ ] High availability basics (replication, Patroni)

**Learning Resources:**
- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Up & Running (O'Reilly)](https://www.oreilly.com/library/view/postgresql-up-and/9781491963401/)
- [Postgres Weekly Newsletter](https://postgresweekly.com/)
- [Cybertec PostgreSQL Blog](https://www.cybertec-postgresql.com/en/blog/)

**Migration Path:**
```yaml
# Why migrate SQLite → PostgreSQL?
Benefits:
  - Better backup options (pg_dump, PITR)
  - Concurrent access (multiple apps/services)
  - Replication (HA/DR scenarios)
  - Better tooling (monitoring, optimization)
  
Trade-offs:
  - More operational complexity
  - Higher resource usage
  - Additional failure point
  - Network dependency
```

---

### Version Control and Collaboration

#### Git
**Required Level:** Intermediate

**Specific Competencies:**
- [ ] Basic operations (clone, add, commit, push, pull)
- [ ] Branching and merging
- [ ] Pull request workflows
- [ ] Resolving conflicts
- [ ] .gitignore patterns
- [ ] Tagging and releases

**Learning Resources:**
- [Pro Git Book (Free)](https://git-scm.com/book/en/v2)
- [Learn Git Branching (Interactive)](https://learngitbranching.js.org/)
- [GitHub Skills](https://skills.github.com/)

**Best Practices for IaC Repos:**
```bash
# .gitignore essentials
*.retry          # Ansible retry files
.vault_pass      # Vault password file
host_vars/**/vault.yml  # Encrypted secrets
.DS_Store        # macOS
__pycache__/     # Python
*.swp            # Vim swap files

# Never commit:
- Passwords/API keys
- Private SSH keys
- Sensitive IP addresses (sometimes)
```

---

## Skills Developed Through This Implementation

### Technical Skills Gained

#### 1. Infrastructure Design Patterns
**What You Learn:**
- Template Method pattern for role abstraction
- Separation of concerns (data vs application vs versions)
- Idempotent operations design
- Declarative vs imperative approaches

**Transferable To:**
- Kubernetes Operator development
- Terraform module creation
- Any IaC platform

**Evidence of Mastery:**
```yaml
# Creating the media_app role demonstrates understanding of:
# - Abstraction layers
# - Parameterization
# - DRY principle
# - Reusability patterns
```

---

#### 2. Security-First Mindset
**What You Learn:**
- Defense in depth (multiple security layers)
- Least privilege principle
- Syscall filtering rationale
- Capability-based security

**Transferable To:**
- Container security (Docker, Podman)
- Kubernetes Pod Security Standards
- Application security reviews
- Compliance audits (SOC2, PCI-DSS)

**Industry Relevance:**
Companies like Google, Facebook, and banks use capability-based security heavily.

---

#### 3. Filesystem Architecture
**What You Learn:**
- Copy-on-write benefits and trade-offs
- Snapshot vs backup strategies
- Storage hierarchy design
- Performance tuning for workloads

**Transferable To:**
- Cloud storage (EBS snapshots, GCS versioning)
- Database storage design
- Backup architecture
- Disaster recovery planning

---

#### 4. Automation Proficiency
**What You Learn:**
- Ansible role development
- Jinja2 templating
- YAML data structures
- Idempotency patterns
- Error handling strategies

**Transferable To:**
- CI/CD pipeline development
- Cloud automation (Terraform, CloudFormation)
- Any automation platform

**Career Impact:**
Configuration management is a core DevOps skill. Companies hiring for:
- DevOps Engineer
- Site Reliability Engineer (SRE)
- Platform Engineer
- Cloud Architect

...all expect Ansible/similar experience.

---

#### 5. Operational Excellence
**What You Learn:**
- Self-healing systems (automated maintenance)
- Monitoring and observability concepts
- Backup and restore procedures
- Disaster recovery planning

**Transferable To:**
- SRE practices
- Production operations
- Incident response
- Business continuity planning

---

### Soft Skills Developed

#### Systems Thinking
**What This Project Teaches:**
- How components interact (systemd → ZFS → apps)
- Cascading failures and dependencies
- Trade-off analysis (security vs performance)
- Holistic problem solving

**Career Relevance:**
Essential for senior engineering roles. Ability to see "the big picture" separates senior from junior engineers.

---

#### Documentation and Knowledge Transfer
**What You Learn:**
- Self-documenting code (good variable names)
- README creation
- Architecture diagrams
- Runbook development

**Why It Matters:**
"Code is read 10x more than it's written." Good documentation:
- Reduces onboarding time
- Prevents single points of knowledge
- Demonstrates professionalism

---

#### Problem Decomposition
**What This Project Teaches:**
- Breaking complex problems into smaller pieces
- Modular design for easier testing
- Incremental delivery (cycles of effort)

**Transfer To:**
- Project management
- Feature development
- Troubleshooting methodology

---

## Skill Gap Analysis

### Current Implementation vs Industry Expectations

| Skill Area | This Project | Industry Standard | Gap |
|------------|--------------|-------------------|-----|
| **IaC Fundamentals** | ✅ Advanced | Advanced | None |
| **Security Hardening** | ✅ Expert | Intermediate | Exceeds |
| **Secrets Management** | ❌ None | Required | Critical |
| **Testing/CI** | ⚠️ Basic (lint only) | Advanced (molecule, CI/CD) | Moderate |
| **Monitoring** | ⚠️ Basic (logs only) | Advanced (metrics, alerts) | Moderate |
| **Documentation** | ⚠️ Minimal | Comprehensive | Moderate |
| **HA/DR** | ⚠️ ZFS snapshots only | Full strategy | Moderate |

**Recommended Learning Path:**
1. **Immediate:** Implement ansible-vault (1-2 hours)
2. **Short-term:** Add Molecule tests (1 week)
3. **Medium-term:** Prometheus/Grafana (2 weeks)
4. **Long-term:** Multi-node HA (1-2 months)

---

## Certification Pathways

### Relevant Certifications

#### Linux Foundation Certified System Administrator (LFCS)
**Why Relevant:** Covers systemd, filesystems, scripting
**Preparation Time:** 40-60 hours
**Cost:** $395
**Link:** [LFCS Exam](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/)

#### Red Hat Certified Specialist in Ansible Automation
**Why Relevant:** Validates Ansible role development skills
**Preparation Time:** 60-80 hours (if familiar with Ansible)
**Cost:** $400
**Link:** [EX374 Exam](https://www.redhat.com/en/services/training/ex374-red-hat-certified-specialist-ansible-automation-exam)

#### Certified Kubernetes Administrator (CKA)
**Why Relevant:** Next step for scaling beyond single-node
**Preparation Time:** 80-120 hours
**Cost:** $395
**Link:** [CKA Exam](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)

---

## Learning Resources by Skill Level

### For This Project Specifically

**Beginner (Getting Started):**
- [Ansible Tutorial for Beginners](https://spacelift.io/blog/ansible-tutorial)
- [Linux Journey](https://linuxjourney.com/)
- [ZFS Quickstart Guide](https://openzfs.github.io/openzfs-docs/Getting%20Started/index.html)

**Intermediate (Understanding Concepts):**
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [systemd for Administrators (Blog Series)](https://www.freedesktop.org/wiki/Software/systemd/)
- [FreeBSD Handbook: ZFS Chapter](https://docs.freebsd.org/en/books/handbook/zfs/)

**Advanced (Deep Dives):**
- [FreeBSD Mastery: ZFS (Book)](https://www.tiltedwindmillpress.com/product/fmzfs/)
- [Linux Kernel Documentation: Security](https://www.kernel.org/doc/html/latest/security/index.html)
- [Site Reliability Engineering (Google SRE Book)](https://sre.google/books/)

---

## Related Notes

- [[Skills Developed Through Implementation]]
- [[Career Development Pathways]]
- [[Industry Standard Comparisons]]

---

**Next Steps:**
1. Self-assess current skill levels
2. Identify 2-3 focus areas for improvement
3. Set learning goals (e.g., "Implement ansible-vault by end of month")
4. Track progress in personal knowledge base
