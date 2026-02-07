---
title: ZFS Integration Patterns
tags: [#zfs, #filesystems, #storage, #advanced]
created: 2026-02-06
---

# ZFS Integration Patterns

## ZFS Fundamentals

### What Makes ZFS Different

ZFS is a **combined filesystem and volume manager** that uses copy-on-write (CoW) architecture:

```
Traditional Stack              ZFS Stack
┌──────────────┐              ┌──────────────┐
│  Filesystem  │              │     ZFS      │
│   (ext4)     │              │  (combined)  │
├──────────────┤              │              │
│    LVM       │              │              │
├──────────────┤              │              │
│  RAID (md)   │              │              │
├──────────────┤              │              │
│  Partitions  │              │              │
├──────────────┤              │              │
│  Block Dev   │              └──────────────┘
└──────────────┘              └──────────────┘
```

**Key Concepts:**

1. **Pool (zpool)** - Collection of physical disks
2. **Dataset** - Logical filesystem (can be nested)
3. **Snapshot** - Read-only point-in-time copy
4. **Clone** - Writable snapshot fork
5. **ZVol** - Block device for VMs

**Reference:**
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)
- [FreeBSD ZFS Primer](https://docs.freebsd.org/en/books/handbook/zfs/)
- [Oracle ZFS Administration Guide](https://docs.oracle.com/cd/E19253-01/819-5461/zfsover-2/)

### Copy-on-Write Explained

When data is modified:

```
Original State:
┌─────────┐
│ Block A │ ← File points here
└─────────┘

Write Operation:
┌─────────┐     ┌─────────┐
│ Block A │     │ Block B │ ← New data written
└─────────┘     └─────────┘
    ↑               ↑
   Old            File now points here

Commit:
┌─────────┐     ┌─────────┐
│ Block A │     │ Block B │
└─────────┘     └─────────┘
    ↑               ↑
 Snapshot        Current
```

**Benefits:**
- No write-in-place (safer)
- Snapshots are "free" (share unchanged blocks)
- Data integrity (checksums on everything)
- No fsck needed (always consistent)

**Reference:**
- [ZFS Copy-on-Write](https://en.wikipedia.org/wiki/ZFS#Copy-on-write_transactional_model)
- [How ZFS Snapshots Work](https://www.oracle.com/technical-resources/articles/it-infrastructure/admin-o11-113-size-zfs-snapshots.html)

## Implementation Patterns for Media Stack

### Pattern 1: Per-App Dataset Strategy

#### Structure

```bash
tank                                    # Pool
├── appdata                            # Parent dataset
│   ├── prowlarr                       # App dataset
│   │   ├── data                       # Data dataset (snapshotted)
│   │   └── versions                   # Versions dataset
│   ├── sonarr
│   │   ├── data
│   │   └── versions
│   └── radarr
│       ├── data
│       └── versions
```

#### Implementation

```yaml
# roles/zfs_datasets/tasks/main.yml
- name: Create app parent dataset
  community.general.zfs:
    name: tank/appdata
    state: present
    extra_zfs_properties:
      compression: lz4
      atime: off
      xattr: sa
      relatime: on

- name: Create per-app datasets
  community.general.zfs:
    name: "tank/appdata/{{ item }}"
    state: present
    extra_zfs_properties:
      quota: 50G                    # Prevent runaway growth
      reservation: 5G               # Guaranteed space
  loop:
    - prowlarr
    - sonarr
    - radarr

- name: Create data subdatasets with snapshot policies
  community.general.zfs:
    name: "tank/appdata/{{ item }}/data"
    state: present
    extra_zfs_properties:
      recordsize: 128k              # Optimal for SQLite
      compression: lz4
      atime: off
      sync: standard                # Balance safety/performance
      logbias: latency              # Prioritize consistency
      snapdir: hidden               # Don't show in listings
      # Snapshot retention policies
      com.sun:auto-snapshot: "true"
      com.sun:auto-snapshot:frequent: "false"
      com.sun:auto-snapshot:hourly: "true"
      com.sun:auto-snapshot:daily: "true"
      com.sun:auto-snapshot:weekly: "true"
      com.sun:auto-snapshot:monthly: "true"
  loop:
    - prowlarr
    - sonarr
    - radarr

- name: Create versions subdatasets
  community.general.zfs:
    name: "tank/appdata/{{ item }}/versions"
    state: present
    extra_zfs_properties:
      recordsize: 1M                # Large sequential reads
      compression: lz4
      atime: off
      sync: disabled                # Performance (safe for binaries)
  loop:
    - prowlarr
    - sonarr
    - radarr
```

#### Why This Pattern?

**Granular Control:**
- Different properties per dataset
- Independent snapshots per app
- Quota enforcement per app
- Easy to move/replicate individual apps

**Example Use Case:**
```bash
# Migrate just Sonarr to new pool
zfs send -R tank/appdata/sonarr | zfs receive new-pool/appdata/sonarr

# Destroy only Prowlarr (with confirmation)
zfs destroy -r tank/appdata/prowlarr
```

**Reference:**
- [ZFS Dataset Hierarchy Best Practices](https://www.solaris-cookbook.eu/solaris/solaris-10-zfs-administration/#zfs-filesystem-hierarchy)

### Pattern 2: Automated Snapshot Lifecycle

#### Pre-Deployment Snapshots

```yaml
# roles/media_app/tasks/snapshot.yml
- name: Create pre-deployment snapshot
  community.general.zfs:
    name: "tank/appdata/{{ media_app_name }}/data@deploy-{{ ansible_date_time.epoch }}"
    state: present
  register: snapshot_result
  failed_when: false

- name: Tag snapshot with metadata
  ansible.builtin.command:
    cmd: >
      zfs set 
      ansible:version={{ media_app_version }}
      ansible:playbook={{ ansible_playbook_name }}
      ansible:user={{ ansible_user_id }}
      tank/appdata/{{ media_app_name }}/data@deploy-{{ ansible_date_time.epoch }}
  when: snapshot_result is succeeded
```

#### Snapshot Retention Policy

```yaml
# roles/zfs_maintenance/tasks/prune_snapshots.yml
- name: List snapshots older than retention period
  ansible.builtin.shell: |
    zfs list -H -o name -t snapshot -S creation tank/appdata/{{ item }}/data | 
    awk -F@ '{ if (NR > {{ snapshot_retention_count | default(30) }}) print $0 }'
  register: old_snapshots
  changed_when: false
  loop:
    - prowlarr
    - sonarr
    - radarr

- name: Destroy old snapshots
  ansible.builtin.command:
    cmd: "zfs destroy {{ item.1 }}"
  loop: "{{ old_snapshots.results | subelements('stdout_lines') }}"
  when: item.1 != ""
```

#### Automated Snapshots via zfs-auto-snapshot

```bash
# Install zfs-auto-snapshot (Ubuntu/Debian)
apt-get install zfs-auto-snapshot

# Cron entries created automatically:
# 15min: */15 * * * * root /usr/sbin/zfs-auto-snapshot frequent  4
# hourly: 0 * * * *   root /usr/sbin/zfs-auto-snapshot hourly   24
# daily:  0 0 * * *   root /usr/sbin/zfs-auto-snapshot daily     7
# weekly: 0 0 * * 0   root /usr/sbin/zfs-auto-snapshot weekly    4
# monthly:0 0 1 * *   root /usr/sbin/zfs-auto-snapshot monthly  12
```

**Retention Math:**
- Hourly (24 snapshots) = 1 day of hourly recovery
- Daily (7 snapshots) = 1 week of daily recovery
- Weekly (4 snapshots) = 1 month of weekly recovery
- Monthly (12 snapshots) = 1 year of monthly recovery

**Storage Impact:**
```bash
# Check snapshot space usage
zfs list -o space -r tank/appdata/prowlarr/data

NAME                           AVAIL   USED  USEDSNAP  USEDDS  USEDREFRESERV  USEDCHILD
tank/appdata/prowlarr/data      450G  1.2G      150M    1.0G              0B         0B
```

Only differences between snapshots consume space (150M for all snapshots).

**Reference:**
- [zfs-auto-snapshot GitHub](https://github.com/zfsonlinux/zfs-auto-snapshot)
- [Snapshot Management Best Practices](https://www.freebsd.org/doc/handbook/zfs-zfs.html)

### Pattern 3: ZFS Send/Receive for Backups

#### Local Replication

```yaml
# playbooks/backup_local.yml
- name: Replicate to backup pool
  hosts: control
  tasks:
    - name: Create snapshot for backup
      community.general.zfs:
        name: "tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }}"
        state: present
      loop:
        - prowlarr
        - sonarr
        - radarr

    - name: Send to backup pool (incremental)
      ansible.builtin.shell: |
        # Find last common snapshot
        LAST=$(zfs list -H -o name -t snapshot backup-pool/{{ item }}/data | tail -1 | cut -d@ -f2)
        
        if [ -n "$LAST" ]; then
          # Incremental send
          zfs send -i tank/appdata/{{ item }}/data@${LAST} \
                      tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }} | \
          zfs receive -F backup-pool/{{ item }}/data
        else
          # Full send (first time)
          zfs send -R tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }} | \
          zfs receive backup-pool/{{ item }}/data
        fi
      loop:
        - prowlarr
        - sonarr
        - radarr
```

#### Remote Replication (Offsite)

```yaml
- name: Replicate to remote server
  ansible.builtin.shell: |
    zfs send -R tank/appdata/{{ item }}/data@backup-{{ ansible_date_time.date }} | \
    ssh backup-server "zfs receive -F backup-pool/homelab/{{ item }}/data"
  loop:
    - prowlarr
    - sonarr
    - radarr
  environment:
    SSH_AUTH_SOCK: "{{ ansible_env.SSH_AUTH_SOCK }}"
```

**Performance Characteristics:**

```bash
# Full send (first time)
zfs send -R tank/appdata/prowlarr/data@backup-2024-01-01
# ~1GB = ~2 minutes on gigabit ethernet

# Incremental send (daily changes)
zfs send -i @yesterday @today tank/appdata/prowlarr/data
# ~50MB = ~6 seconds
```

**Compression in Transit:**
```bash
# Compress over network
zfs send tank/appdata/prowlarr/data@backup | \
  gzip | \
  ssh backup-server "gunzip | zfs receive backup-pool/prowlarr/data"

# Even better: use zstd
zfs send tank/appdata/prowlarr/data@backup | \
  zstd -3 | \
  ssh backup-server "zstd -d | zfs receive backup-pool/prowlarr/data"
```

**Reference:**
- [ZFS Send/Receive Guide](https://openzfs.github.io/openzfs-docs/man/8/zfs-send.8.html)
- [Syncoid (Advanced Replication)](https://github.com/jimsalterjrs/sanoid)

### Pattern 4: Dataset Property Tuning

#### SQLite Optimization

```bash
# Current state (SQLite databases)
zfs set recordsize=128k tank/appdata/prowlarr/data
zfs set primarycache=metadata tank/appdata/prowlarr/data
zfs set logbias=latency tank/appdata/prowlarr/data
zfs set sync=standard tank/appdata/prowlarr/data
```

**Why These Values:**

| Property | Value | Reason |
|----------|-------|--------|
| `recordsize=128k` | SQLite page size (typically 4k-32k) | Reduces read amplification |
| `primarycache=metadata` | Only cache metadata in ARC | More RAM for data cache |
| `logbias=latency` | Prefer latency over throughput | SQLite is latency-sensitive |
| `sync=standard` | fsync() honored | Prevents database corruption |

**Benchmarking:**
```bash
# Before tuning
fio --name=sqlite-sim --size=1G --rw=randrw --bs=4k --direct=1 --runtime=60

# After tuning (expect 20-40% improvement in random I/O)
```

**Reference:**
- [ZFS Tuning for Databases](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html#database)
- [SQLite on ZFS](https://www.sqlite.org/zfs.html)

#### PostgreSQL Optimization (Future)

```bash
zfs create tank/postgres
zfs set recordsize=8k tank/postgres/data
zfs set logbias=latency tank/postgres/data
zfs set primarycache=metadata tank/postgres/data
zfs set compression=lz4 tank/postgres/data
zfs set atime=off tank/postgres/data
zfs set redundant_metadata=most tank/postgres/data
```

**Why 8k recordsize:**
- PostgreSQL default page size = 8KB
- Matching recordsize prevents read/write amplification
- One ZFS block = One PostgreSQL page

**Expected Performance:**
- 30-50% better random read performance
- 15-25% better write throughput
- Lower write amplification (important for SSD lifespan)

**Reference:**
- [PostgreSQL on ZFS Best Practices](https://wiki.postgresql.org/wiki/ZFS)
- [ZFS for PostgreSQL (2ndQuadrant)](https://www.2ndquadrant.com/en/blog/postgresql-zfs-best-practices/)

### Pattern 5: Disaster Recovery Workflows

#### Full System Restore

```yaml
# playbooks/disaster_recovery.yml
- name: Restore from ZFS snapshots
  hosts: control
  vars_prompt:
    - name: snapshot_name
      prompt: "Snapshot to restore (e.g., @backup-2024-01-01)"
      private: false
  
  tasks:
    - name: Stop all services
      ansible.builtin.systemd:
        name: "{{ item }}.service"
        state: stopped
      loop:
        - prowlarr
        - sonarr
        - radarr

    - name: Rollback to snapshot
      ansible.builtin.command:
        cmd: "zfs rollback -r tank/appdata/{{ item }}/data{{ snapshot_name }}"
      loop:
        - prowlarr
        - sonarr
        - radarr

    - name: Start all services
      ansible.builtin.systemd:
        name: "{{ item }}.service"
        state: started
      loop:
        - prowlarr
        - sonarr
        - radarr

    - name: Verify services
      ansible.builtin.uri:
        url: "http://localhost:{{ item.port }}/ping"
        status_code: 200
      loop:
        - { name: prowlarr, port: 9696 }
        - { name: sonarr, port: 8989 }
        - { name: radarr, port: 7878 }
      retries: 5
      delay: 10
```

#### Selective File Restore

```bash
# User accidentally deleted config file
# Don't rollback entire dataset, just restore one file

# List snapshots
zfs list -t snapshot tank/appdata/prowlarr/data

# Mount snapshot read-only
zfs mount tank/appdata/prowlarr/data@hourly-2024-01-01-12:00

# Copy file from snapshot
cp /tank/appdata/prowlarr/data/.zfs/snapshot/hourly-2024-01-01-12:00/config.xml \
   /tank/appdata/prowlarr/data/config.xml

# Unmount snapshot (automatic after access ends)
```

**Reference:**
- [ZFS Disaster Recovery](https://pthree.org/2012/12/18/zfs-administration-part-xii-snapshots-and-clones/)

## Advanced Patterns

### Deduplication (Use with Caution)

```bash
# Enable dedup on versions dataset (similar binaries)
zfs set dedup=on tank/appdata/prowlarr/versions
```

**WARNING:** Deduplication requires ~5GB RAM per 1TB of deduplicated data.

**Better Alternative:** Compression achieves similar results without RAM penalty.

**Reference:**
- [ZFS Deduplication Analysis](https://blog.ksplice.com/2011/05/zfs-deduplication-and-its-cost/)

### Encryption at Rest

```bash
# Create encrypted dataset (requires ZFS 0.8.0+)
zfs create -o encryption=aes-256-gcm \
           -o keyformat=passphrase \
           tank/appdata-encrypted

# Mount requires passphrase
zfs mount -l tank/appdata-encrypted
```

**Use Cases:**
- Offsite backups (untrusted storage)
- Compliance requirements
- Sensitive data protection

**Reference:**
- [ZFS Native Encryption](https://openzfs.github.io/openzfs-docs/man/8/zfs-load-key.8.html)

## Monitoring and Observability

### Health Checks

```yaml
# roles/zfs_monitoring/tasks/health.yml
- name: Check pool health
  ansible.builtin.command: zpool status tank
  register: zpool_health
  changed_when: false
  failed_when: "'DEGRADED' in zpool_health.stdout or 'FAULTED' in zpool_health.stdout"

- name: Check for scrub errors
  ansible.builtin.shell: zpool status tank | grep "errors:" | grep -v "No known data errors"
  register: scrub_errors
  changed_when: false
  failed_when: scrub_errors.rc == 0

- name: Get pool capacity
  ansible.builtin.shell: zpool list -H -o capacity tank | sed 's/%//'
  register: pool_capacity
  changed_when: false
  failed_when: pool_capacity.stdout | int > 80
```

### Prometheus Exporter

```yaml
# Install ZFS exporter for metrics
- name: Install ZFS Prometheus exporter
  ansible.builtin.get_url:
    url: https://github.com/pdf/zfs_exporter/releases/download/v2.2.5/zfs_exporter-2.2.5.linux-amd64.tar.gz
    dest: /tmp/zfs_exporter.tar.gz

- name: Extract and install
  ansible.builtin.unarchive:
    src: /tmp/zfs_exporter.tar.gz
    dest: /usr/local/bin/
    remote_src: yes

- name: Create systemd service
  ansible.builtin.copy:
    dest: /etc/systemd/system/zfs-exporter.service
    content: |
      [Unit]
      Description=ZFS Prometheus Exporter
      After=network.target

      [Service]
      Type=simple
      ExecStart=/usr/local/bin/zfs_exporter
      Restart=always

      [Install]
      WantedBy=multi-user.target
```

**Metrics Exposed:**
- Pool capacity and health
- Dataset sizes and snapshot counts
- ARC hit/miss ratios
- I/O statistics
- Scrub status

**Reference:**
- [ZFS Exporter GitHub](https://github.com/pdf/zfs_exporter)

## Case Studies

### Netflix CDN (Open Connect)

Netflix uses ZFS for their CDN appliances:
- Stores 100TB+ per server
- Leverages ARC for hot content
- Uses snapshots for rapid updates
- Achieves 40Gbps+ throughput per node

**Reference:**
- [Netflix Open Connect Appliance](https://openconnect.netflix.com/en/hardware/)

### iXsystems TrueNAS

TrueNAS uses ZFS as primary storage layer:
- Powers enterprise storage solutions
- Millions of deployments worldwide
- Demonstrates ZFS reliability at scale

**Reference:**
- [TrueNAS Documentation](https://www.truenas.com/docs/)

## Related Notes

- [[Technical Architecture Overview]]
- [[Database Architecture Evolution]]
- [[Backup and Disaster Recovery]]

---

**Further Reading:**
- [FreeBSD Mastery: ZFS (Michael W Lucas)](https://www.tiltedwindmillpress.com/product/fmzfs/)
- [ZFS Quickstart Guide](https://openzfs.github.io/openzfs-docs/Getting%20Started/index.html)
