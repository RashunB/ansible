---
title: Technical Architecture Overview
tags: [#architecture, #design-patterns, #infrastructure]
created: 2026-02-06
---

# Technical Architecture Overview

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      ZFS Pool (tank)                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              tank/appdata (Dataset)                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │   │
│  │  │ prowlarr/    │  │ sonarr/      │  │ radarr/  │  │   │
│  │  │ ├─ data/     │  │ ├─ data/     │  │ ├─ data/ │  │   │
│  │  │ └─ versions/ │  │ └─ versions/ │  │ └─versions│  │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘  │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │         tank/postgres/data (Future)          │  │   │
│  │  │  ├─ prowlarr_db                              │  │   │
│  │  │  ├─ sonarr_db                                │  │   │
│  │  │  └─ radarr_db                                │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Systemd Services Layer                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  prowlarr.service  │  sonarr.service │ radarr.service│  │
│  │  ─────────────────────────────────────────────────── │  │
│  │  • ProtectSystem=strict                              │  │
│  │  • CapabilityBoundingSet=                            │  │
│  │  • SystemCallFilter=@system-service                  │  │
│  │  • BindReadOnlyPaths (app binaries)                  │  │
│  │  • BindPaths (data directories)                      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Ansible Control Plane                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Playbooks:                                           │  │
│  │  • controller.yml  - Main deployment                 │  │
│  │  • maintenance.yml - Automated updates               │  │
│  │  • bootstrap.yml   - Initial setup                   │  │
│  │                                                       │  │
│  │  Roles:                                               │  │
│  │  • common         - Base system config               │  │
│  │  • media_app      - Abstraction layer                │  │
│  │  • prowlarr/sonarr/radarr - App-specific params      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Core Design Patterns

### 1. Template Method Pattern (media_app Role)

The `media_app` role implements a template method pattern where the workflow is fixed, but individual steps can be customized:

```yaml
# Template (roles/media_app/tasks/main.yml)
- packages    # Customizable via media_app_packages
- users       # Standard user creation
- directories # Customizable via media_app_app_srv
- download    # Parametrized download
- link        # Atomic symlink deployment
- service     # Template-based systemd unit
```

**Industry Parallel:** 
- Kubernetes Operators (fixed reconciliation loop, custom resources)
- Terraform Modules (standardized interface, variable backends)

**Reference:**
- [Design Patterns in Infrastructure Code](https://www.hashicorp.com/resources/terraform-design-patterns-the-ultimate-guide)
- [Ansible Role Best Practices](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)

### 2. Blue-Green Deployment via Symlinks

```bash
# Directory structure
/tank/appdata/prowlarr/
├── versions/
│   ├── prowlarr_2.0.5.5160/    # Current
│   └── prowlarr_2.0.4.5000/    # Previous (retained)
└── app → versions/prowlarr_2.0.5.5160  # Symlink
```

**Deployment Process:**
1. Download new version to `/versions/`
2. Extract and verify
3. Atomically update symlink
4. Restart service (reads new binary)

**Rollback:** Just repoint symlink and restart

**Enterprise Equivalent:**
- Kubernetes Rolling Updates (but with instant rollback)
- AWS CodeDeploy Blue/Green
- Capistrano releases pattern

**Reference:**
- [Atomic Operations in Unix](https://lwn.net/Articles/589978/)
- [Blue-Green Deployments (Martin Fowler)](https://martinfowler.com/bliki/BlueGreenDeployment.html)

### 3. Bind Mounts for Filesystem Isolation

```systemd
# Service runs in /srv namespace
WorkingDirectory=/srv/prowlarr

# Actual files on ZFS
BindReadOnlyPaths=/tank/appdata/prowlarr/app:/srv/prowlarr/app
BindPaths=/tank/appdata/prowlarr/data:/srv/prowlarr/data
```

**Security Benefits:**
- App cannot see real filesystem paths
- Cannot escape to parent directories
- Read-only binaries prevent self-modification
- Contained blast radius if compromised

**Industry Standard:**
- Container bind mounts (Docker `-v` flag)
- chroot jails (but better via namespaces)
- SELinux/AppArmor policies (complementary)

**Reference:**
- [systemd Bind Mounts](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#BindPaths=)
- [Linux Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)

### 4. Copy-on-Write (CoW) Snapshots

ZFS enables instant, space-efficient snapshots:

```bash
# Before deployment
zfs snapshot tank/appdata/prowlarr/data@pre-update-1707264000

# If deployment fails
zfs rollback tank/appdata/prowlarr/data@pre-update-1707264000

# Space usage (only differences stored)
zfs list -t snapshot
NAME                                           USED
tank/appdata/prowlarr/data@pre-update          42K
```

**Enterprise Equivalents:**
- NetApp WAFL snapshots
- AWS EBS snapshots (but not instant)
- Kubernetes VolumeSnapshots (CSI drivers)
- VMware Storage vMotion

**Reference:**
- [OpenZFS Snapshots](https://openzfs.github.io/openzfs-docs/man/8/zfs-snapshot.8.html)
- [Copy-on-Write Explained](https://en.wikipedia.org/wiki/Copy-on-write)

## Technical Benefits Analysis

### 1. **Idempotency**

Every Ansible task can be run repeatedly without side effects:

```yaml
# This is safe to run 100 times
- name: Create symbolic link
  ansible.builtin.file:
    src: "{{ media_app_versions }}/{{ version }}"
    dest: "{{ media_app_app }}"
    state: link
    force: true
```

**Why This Matters:**
- Safe to re-run deployments after partial failures
- Configuration drift detection and correction
- Consistent state regardless of starting point

**Industry Standard:**
- Terraform `apply` (but with plan preview)
- Kubernetes reconciliation loops
- Chef/Puppet convergence

**Reference:**
- [Idempotence in Automation](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html#term-Idempotency)
- [Declarative vs Imperative IaC](https://www.terraform.io/intro/vs/chef-puppet)

### 2. **Atomic Operations**

Multiple operations appear as a single, indivisible unit:

```yaml
# Symlink update + service restart = atomic deployment
- file: state=link ...
  notify: Restart App
  
# Handler runs once at end, not per-task
handlers:
  - name: Restart App
    systemd: state=restarted
```

**Guarantees:**
- Service never sees half-updated state
- Either fully old or fully new version
- No torn reads/writes

**Reference:**
- [ACID Properties](https://en.wikipedia.org/wiki/ACID)
- [Ansible Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)

### 3. **Fail-Safe Defaults**

Security restrictions are default, access is granted explicitly:

```systemd
# Deny everything
ProtectSystem=strict
CapabilityBoundingSet=

# Explicitly allow only what's needed
BindPaths=/tank/appdata/prowlarr/data:/srv/prowlarr/data
SystemCallFilter=@system-service @network-io
```

**Security Principle:** Whitelist, not blacklist

**Reference:**
- [Principle of Least Privilege](https://csrc.nist.gov/glossary/term/least_privilege)
- [systemd Security Features](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Security)

### 4. **Version Control for Infrastructure**

All configuration in Git repository:

```bash
ansible-main/
├── .git/                    # Full history
├── playbooks/
├── roles/
└── inventories/
```

**Benefits:**
- Audit trail (who changed what, when, why)
- Peer review via pull requests
- Rollback to any previous state
- Disaster recovery (clone repo, run playbook)

**Industry Standard:**
- GitOps (Flux, ArgoCD for Kubernetes)
- Terraform Cloud (state management)
- AWS CloudFormation (StackSets)

**Reference:**
- [GitOps Principles](https://opengitops.dev/)
- [Infrastructure as Code Book (Kief Morris)](https://www.oreilly.com/library/view/infrastructure-as-code/9781098114664/)

## Performance Characteristics

### ZFS + Ansible Optimizations

| Aspect | Implementation | Benefit |
|--------|---------------|---------|
| **Fact Caching** | `fact_caching_timeout = 86400` | Reduces SSH overhead, 24h TTL |
| **SSH Pipelining** | `pipelining = True` | Fewer SSH connections, faster execution |
| **Smart Gathering** | `gathering = smart` | Only refresh facts when needed |
| **ZFS Compression** | `compression=lz4` (assumed) | ~2-3x capacity increase, negligible CPU |
| **ZFS ARC** | Metadata caching | Hot data in RAM, faster reads |
| **Recordsize Tuning** | 128K for SQLite | Matches workload pattern |

**Benchmark Context:**
- Ansible playbook runs: ~30-60 seconds (vs hours for manual)
- ZFS snapshots: <1 second (vs minutes for rsync)
- Rollback: ~5 seconds (service restart time)
- Compression ratio: 1.5x - 3.0x (typical for media metadata)

**Reference:**
- [Ansible Performance Tuning](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-speed-up-playbook-execution)
- [OpenZFS Tuning Guide](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html)

## Scalability Analysis

### Current Limitations

1. **Single-Node Architecture**
   - No high availability
   - Maintenance requires downtime
   - Bounded by single server resources

2. **Local-Only Storage**
   - ZFS pool on single machine
   - No distributed storage (yet)

3. **Manual Coordination**
   - Service dependencies managed via systemd
   - No service mesh or orchestration

### Scaling Paths

**Horizontal Scaling (Future):**
```yaml
# Multi-host inventory
[media_servers]
media-01 ansible_host=192.168.1.10
media-02 ansible_host=192.168.1.11

[databases]
postgres-01 ansible_host=192.168.1.20
```

**Vertical Scaling (Current):**
- More RAM = Larger ZFS ARC = Better performance
- More CPU = Faster transcoding (if Plex added)
- More disks = RAID-Z2/Z3, more IOPS

**Enterprise Pattern:**
- Kubernetes (horizontal pod autoscaling)
- Load balancers (HAProxy, Nginx)
- Distributed databases (PostgreSQL HA with Patroni)

**Reference:**
- [Scaling Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html)
- [ZFS RAID Levels](https://openzfs.github.io/openzfs-docs/man/7/zpoolconcepts.7.html)

## Architectural Trade-offs

### What Was Chosen & Why

| Decision | Alternative | Trade-off |
|----------|------------|-----------|
| **Ansible** | Docker Compose | More control, better security, steeper learning |
| **Systemd** | Docker | Native integration, less overhead, harder debugging |
| **ZFS** | ext4/XFS | Snapshots/compression, but higher RAM usage |
| **SQLite → PostgreSQL** | Keep SQLite | Better for backups/HA, more operational complexity |
| **Binary Downloads** | Package Manager | Latest versions, manual version management |
| **Local Execution** | Remote | Simpler, but less flexible for multi-node |

### Technical Debt Recognized

1. **Hardcoded Paths** - Limits portability
2. **No Secrets Management** - Security gap
3. **Missing Observability** - Operational blind spots
4. **No Automated Tests** - Quality assurance gap

**Industry Lesson:** Technical debt is normal; acknowledging and planning for it is what matters.

**Reference:**
- [Technical Debt Quadrant (Martin Fowler)](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html)

## Related Notes

- [[ZFS Integration Patterns]]
- [[Systemd Security Hardening]]
- [[Industry Standard Comparisons]]
- [[Database Architecture Evolution]]

---

**See Also:**
- [Linux Application Security](https://www.kernel.org/doc/html/latest/admin-guide/LSM/index.html)
- [Site Reliability Engineering Book (Google)](https://sre.google/books/)
