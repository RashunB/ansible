---
title: Database Architecture Evolution
tags: [#database, #postgresql, #sqlite, #migration, #architecture]
created: 2026-02-06
---

# Database Architecture Evolution

## Current State: SQLite

### Architecture

```
┌──────────────────────────────────────┐
│       Application (Prowlarr)         │
│  ┌────────────────────────────────┐  │
│  │  SQLite Library (embedded)     │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │  prowlarr.db (file)      │  │  │
│  │  │  logs.db (file)          │  │  │
│  │  └──────────────────────────┘  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
         │
         ▼
/tank/appdata/prowlarr/data/prowlarr.db
```

**Key Characteristics:**
- **Embedded database** - No separate server process
- **Single file** - Entire database in one file
- **Local access only** - No network interface
- **Single writer** - One write transaction at a time
- **Read-only while writing** - Readers block on writes (default mode)

**Reference:**
- [SQLite Architecture](https://www.sqlite.org/arch.html)
- [When to Use SQLite](https://www.sqlite.org/whentouse.html)

### Why SQLite Works for Current State

#### Advantages for Media Stack

1. **Zero Administration**
```bash
# No setup required
# Database file created automatically on first run
/tank/appdata/prowlarr/data/prowlarr.db
```

2. **Simple Backups**
```bash
# Backup is just copying a file
cp /tank/appdata/prowlarr/data/prowlarr.db /backup/

# With ZFS
zfs snapshot tank/appdata/prowlarr/data@backup
```

3. **Low Resource Usage**
```bash
# No daemon process
# No persistent connections
# Memory used only when app is active
```

4. **Sufficient Performance**
```sql
-- Typical workload for *arr apps
-- Read-heavy (90% reads, 10% writes)
-- Small transactions
-- Low concurrency

-- Example: Check for new releases
SELECT * FROM indexers WHERE enabled = 1;

-- Write: Update download status
UPDATE downloads SET status = 'completed' WHERE id = 123;
```

#### Current Performance Characteristics

**Benchmarks (Typical Homelab):**
```bash
# Random reads: ~50,000 ops/sec
# Random writes: ~10,000 ops/sec
# Sequential reads: ~100 MB/s
# Sequential writes: ~50 MB/s

# For comparison:
# Your workload: ~100 operations/minute
# Utilization: <1% of capacity
```

**Reference:**
- [SQLite Performance Tuning](https://www.sqlite.org/optoverview.html)

### SQLite Limitations (Why Migrate)

#### 1. Concurrency Constraints

```
┌──────────────────────────────────────┐
│  Multiple Clients (Future)           │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │ App 1  │  │ App 2  │  │ Backup │ │
│  └────────┘  └────────┘  └────────┘ │
│       │           │           │      │
│       └───────────┼───────────┘      │
│                   ▼                  │
│            ┌──────────┐              │
│            │ SQLite   │              │
│            │ (locked) │              │
│            └──────────┘              │
└──────────────────────────────────────┘

Problem: Only one writer at a time
Impact: Backups block application
        Application blocks backups
```

**Real-World Scenario:**
```bash
# Start backup
sqlite3 prowlarr.db '.backup /backup/prowlarr.db'

# Meanwhile, app tries to write
# -> SQLITE_BUSY error
# -> Application retry logic kicks in
# -> User sees "Database is locked" errors
```

#### 2. No Network Access

```
Cannot do this:
┌──────────────┐     Network      ┌──────────────┐
│  App Server  │ ───────────────> │ DB Server    │
│  (Docker)    │                  │ (SQLite)     │
└──────────────┘                  └──────────────┘
                                       ❌

SQLite requires local file access
```

**Implication:** Cannot separate application and database tiers.

#### 3. Limited Replication

```
No native replication:

Primary ────X───> Standby
(SQLite)         (SQLite)

Must use filesystem replication:
- ZFS send/receive (async, manual)
- rsync (not real-time)
- Litestream (third-party tool)
```

**Reference:**
- [Litestream - SQLite Replication](https://litestream.io/)

#### 4. Backup Challenges

```sql
-- Hot backup requires special handling
.backup /backup/db.db

-- Or use ZFS snapshot (your approach)
zfs snapshot tank/appdata/prowlarr/data@backup

-- But no point-in-time recovery (PITR)
-- Cannot replay transactions
```

---

## Target State: PostgreSQL

### Architecture

```
┌─────────────────────────────────────────────────┐
│            Applications Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Prowlarr │  │ Sonarr   │  │ Radarr   │      │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘      │
│        │             │              │            │
│        └─────────────┼──────────────┘            │
│                      │ Network (TCP)             │
└──────────────────────┼───────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│       PostgreSQL Server Process                  │
│  ┌───────────────────────────────────────────┐  │
│  │  Connection Pool (max_connections=100)    │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐   │  │
│  │  │ Backend │  │ Backend │  │ Backend │   │  │
│  │  │ Process │  │ Process │  │ Process │   │  │
│  │  └─────────┘  └─────────┘  └─────────┘   │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │           Shared Buffers (RAM)            │  │
│  └───────────────────────────────────────────┘  │
│                      │                           │
└──────────────────────┼───────────────────────────┘
                       ▼
/tank/postgres/data/
├── base/
│   ├── prowlarr_db/
│   ├── sonarr_db/
│   └── radarr_db/
└── pg_wal/
```

**Key Differences from SQLite:**
- **Client-server** - Separate database process
- **Network protocol** - Apps connect via TCP
- **Concurrent writes** - MVCC (Multi-Version Concurrency Control)
- **Multiple databases** - Logical separation per app
- **Connection pooling** - Reusable connections

**Reference:**
- [PostgreSQL Architecture](https://www.postgresql.org/docs/current/tutorial-arch.html)
- [MVCC Explained](https://www.postgresql.org/docs/current/mvcc-intro.html)

### Benefits of PostgreSQL

#### 1. True Concurrency (MVCC)

```
Multiple Clients Working Simultaneously:

App 1 (Read)  ────────────┐
                          │
App 2 (Write) ────────────┼──> PostgreSQL Server
                          │    (handles concurrency)
Backup (Read) ────────────┘

No locking between readers and writers!
```

**How MVCC Works:**
```sql
-- Transaction 1 starts
BEGIN;
SELECT * FROM releases WHERE id = 1;
-- Sees version 1 of the row

-- Meanwhile, Transaction 2 writes
BEGIN;
UPDATE releases SET status = 'downloaded' WHERE id = 1;
COMMIT;
-- Creates version 2 of the row

-- Transaction 1 still sees version 1
-- No blocking, no waiting
```

**Reference:**
- [PostgreSQL MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)

#### 2. Network Accessibility

```
Deployment Flexibility:

┌──────────────┐        ┌──────────────┐
│  App Server  │        │  DB Server   │
│  (Docker)    │───────>│ (PostgreSQL) │
└──────────────┘ TCP    └──────────────┘
     │                        │
     │                        │
     ▼                        ▼
Can scale                Can optimize
independently            for database
```

**Example: Future Multi-Node Setup**
```yaml
# Separate database server
[databases]
postgres-01 ansible_host=192.168.1.20

[apps]
media-01 ansible_host=192.168.1.10
media-02 ansible_host=192.168.1.11

# Both media servers connect to same database
postgresql_host: postgres-01
```

#### 3. Advanced Backup Capabilities

**pg_dump (Logical Backup)**
```bash
# Backup single database
pg_dump -Fc prowlarr_db > prowlarr_db.dump

# Backup all databases
pg_dumpall > all_databases.sql

# Custom format (compressed, selective restore)
pg_dump -Fc -f prowlarr.dump prowlarr_db
```

**pg_basebackup (Physical Backup)**
```bash
# Full cluster backup
pg_basebackup -D /backup/postgres -Ft -z -P

# Can be used for replication setup
```

**Point-in-Time Recovery (PITR)**
```bash
# Restore to specific timestamp
# "Undo that accidental DELETE 5 minutes ago"

# 1. Restore base backup
# 2. Replay WAL files until target time
recovery_target_time = '2024-01-01 15:30:00'
```

**Reference:**
- [PostgreSQL Backup Guide](https://www.postgresql.org/docs/current/backup.html)

#### 4. Streaming Replication

```
┌──────────────┐         WAL Stream        ┌──────────────┐
│   Primary    │ ──────────────────────> │   Standby    │
│ (Read/Write) │                          │ (Read-Only)  │
└──────────────┘                          └──────────────┘
     │                                          │
     │ Synchronous or Asynchronous              │
     └──────────────────────────────────────────┘

Automatic Failover:
- Primary fails
- Standby promoted to primary
- Applications reconnect
- Zero data loss (synchronous mode)
```

**Implementation:**
```yaml
# roles/postgres_ha/tasks/main.yml
- name: Configure primary for replication
  ansible.builtin.lineinfile:
    path: /tank/postgres/data/postgresql.conf
    line: "{{ item }}"
  loop:
    - "wal_level = replica"
    - "max_wal_senders = 3"
    - "wal_keep_size = 1GB"

- name: Set up replication slot
  community.postgresql.postgresql_slot:
    name: standby_slot
    db: postgres
```

**Reference:**
- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)

#### 5. Performance at Scale

**Query Optimization:**
```sql
-- PostgreSQL query planner
EXPLAIN ANALYZE 
SELECT * FROM releases 
WHERE indexer_id = 1 
  AND published_date > '2024-01-01'
ORDER BY published_date DESC
LIMIT 100;

-- Shows execution plan, index usage, actual runtime
```

**Indexing Strategies:**
```sql
-- B-tree (default, most common)
CREATE INDEX idx_releases_date ON releases(published_date);

-- Partial index (smaller, faster)
CREATE INDEX idx_active_releases 
ON releases(published_date) 
WHERE status = 'active';

-- Multi-column index
CREATE INDEX idx_releases_composite 
ON releases(indexer_id, published_date);
```

**Connection Pooling (PgBouncer):**
```
Without pooling:
Each request = New connection = Expensive

With pooling:
Requests share connections = Fast
```

**Reference:**
- [Use The Index, Luke!](https://use-the-index-luke.com/)
- [PgBouncer](https://www.pgbouncer.org/)

---

## Migration Strategy

### Pre-Migration: Assessment

#### 1. Database Schema Analysis

```bash
# Export SQLite schema
sqlite3 prowlarr.db .schema > schema.sql

# Analyze tables
sqlite3 prowlarr.db << EOF
.tables
.schema indexers
.schema releases
SELECT COUNT(*) FROM releases;
EOF
```

**Expected Findings:**
- ~10-20 tables per app
- Releases table (largest, millions of rows)
- Indexers, downloads, history tables
- Mostly integer primary keys

#### 2. Data Volume Assessment

```bash
# Check database sizes
du -sh /tank/appdata/*/data/*.db

# Expected:
prowlarr.db: 100-500 MB
sonarr.db:   500 MB - 2 GB
radarr.db:   500 MB - 2 GB
```

#### 3. Compatibility Check

```sql
-- SQLite uses some non-standard features
-- Check for:

-- AUTOINCREMENT (PostgreSQL: SERIAL)
CREATE TABLE test (
  id INTEGER PRIMARY KEY AUTOINCREMENT
);

-- DATETIME (PostgreSQL: TIMESTAMP)
CREATE TABLE events (
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Migration Approaches

#### Approach 1: Application-Assisted Migration (Recommended)

```yaml
# playbooks/migrate_to_postgres.yml
- name: Migrate Database
  hosts: control
  vars:
    apps: [prowlarr, sonarr, radarr]
  
  tasks:
    # 1. Install PostgreSQL
    - name: Install PostgreSQL
      ansible.builtin.include_role:
        name: postgres
    
    # 2. Create databases
    - name: Create app databases
      community.postgresql.postgresql_db:
        name: "{{ item }}_db"
        state: present
      loop: "{{ apps }}"
    
    # 3. Stop applications
    - name: Stop apps
      ansible.builtin.systemd:
        name: "{{ item }}.service"
        state: stopped
      loop: "{{ apps }}"
    
    # 4. Snapshot SQLite data
    - name: Create pre-migration snapshot
      community.general.zfs:
        name: "tank/appdata/{{ item }}/data@pre-postgres"
        state: present
      loop: "{{ apps }}"
    
    # 5. Update app config for PostgreSQL
    - name: Configure app for PostgreSQL
      ansible.builtin.template:
        src: config.xml.j2
        dest: "/tank/appdata/{{ item }}/data/config.xml"
      loop: "{{ apps }}"
    
    # 6. Start apps (they handle migration internally)
    - name: Start apps
      ansible.builtin.systemd:
        name: "{{ item }}.service"
        state: started
      loop: "{{ apps }}"
    
    # 7. Verify migration
    - name: Wait for apps to start
      ansible.builtin.wait_for:
        port: "{{ app_ports[item] }}"
        timeout: 120
      loop: "{{ apps }}"
```

**Pros:**
- Apps handle schema creation
- Built-in migration logic
- Preserves application-specific data structures

**Cons:**
- Requires app restart
- Migration time depends on data volume

**Reference:**
- [Servarr PostgreSQL Migration](https://wiki.servarr.com/postgres-setup)

#### Approach 2: pgloader (Alternative)

```bash
# Install pgloader
apt-get install pgloader

# Migrate SQLite to PostgreSQL
pgloader --verbose \
  /tank/appdata/prowlarr/data/prowlarr.db \
  postgresql://prowlarr:password@localhost/prowlarr_db
```

**Pros:**
- Automated schema conversion
- Handles data type mapping
- Progress reporting

**Cons:**
- May not handle app-specific migrations
- Requires manual verification

**Reference:**
- [pgloader Documentation](https://pgloader.readthedocs.io/)

### Post-Migration: Optimization

#### 1. Analyze and Vacuum

```sql
-- Update statistics for query planner
ANALYZE;

-- Reclaim space and update indexes
VACUUM ANALYZE;

-- Auto-vacuum (background process)
ALTER DATABASE prowlarr_db SET autovacuum = on;
```

#### 2. Index Creation

```sql
-- Identify missing indexes
-- Run queries, check EXPLAIN output

-- Common pattern:
CREATE INDEX idx_releases_indexer_date 
ON releases(indexer_id, published_date DESC);
```

#### 3. Connection Tuning

```ini
# /tank/postgres/data/postgresql.conf

# Memory (for 8GB system)
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 64MB
maintenance_work_mem = 512MB

# Connections
max_connections = 100

# Performance
random_page_cost = 1.1  # For SSD/ZFS
effective_io_concurrency = 200
```

**Reference:**
- [PgTune - Configuration Generator](https://pgtune.leopard.in.ua/)

---

## Ansible Implementation

### Role: postgres

```yaml
# roles/postgres/tasks/main.yml
- name: Create PostgreSQL dataset
  community.general.zfs:
    name: tank/postgres/data
    state: present
    extra_zfs_properties:
      recordsize: 8k
      compression: lz4
      logbias: latency
      primarycache: metadata

- name: Install PostgreSQL
  ansible.builtin.package:
    name:
      - postgresql-16
      - postgresql-contrib-16
      - python3-psycopg2
    state: present

- name: Initialize cluster
  ansible.builtin.command:
    cmd: /usr/lib/postgresql/16/bin/initdb -D /tank/postgres/data
    creates: /tank/postgres/data/PG_VERSION
  become_user: postgres

- name: Configure PostgreSQL
  ansible.builtin.template:
    src: postgresql.conf.j2
    dest: /tank/postgres/data/postgresql.conf
  notify: Restart PostgreSQL

- name: Create app databases
  community.postgresql.postgresql_db:
    name: "{{ item }}_db"
    state: present
  loop: "{{ media_apps }}"
  become_user: postgres

- name: Create app users
  community.postgresql.postgresql_user:
    name: "{{ item }}"
    password: "{{ vault_db_passwords[item] }}"
    db: "{{ item }}_db"
    priv: ALL
  loop: "{{ media_apps }}"
  become_user: postgres
```

### Updated media_app Role

```yaml
# roles/media_app/defaults/main.yml
media_app_db_type: postgres  # Changed from 'sqlite'
media_app_db_host: localhost
media_app_db_port: 5432
media_app_db_name: "{{ media_app_name }}_db"
media_app_db_user: "{{ media_app_name }}"
media_app_db_password: "{{ vault_db_passwords[media_app_name] }}"
```

```yaml
# roles/media_app/templates/config.xml.j2
<Config>
  <PostgresHost>{{ media_app_db_host }}</PostgresHost>
  <PostgresPort>{{ media_app_db_port }}</PostgresPort>
  <PostgresDatabase>{{ media_app_db_name }}</PostgresDatabase>
  <PostgresUser>{{ media_app_db_user }}</PostgresUser>
  <PostgresPassword>{{ media_app_db_password }}</PostgresPassword>
</Config>
```

---

## Backup Strategy Comparison

### SQLite (Current)

```bash
# Approach 1: ZFS snapshot
zfs snapshot tank/appdata/prowlarr/data@daily

# Approach 2: File copy
cp prowlarr.db prowlarr.db.backup

# Approach 3: SQLite backup command
sqlite3 prowlarr.db ".backup /backup/prowlarr.db"
```

**Recovery:**
```bash
# Rollback snapshot
zfs rollback tank/appdata/prowlarr/data@daily

# Or restore file
cp prowlarr.db.backup prowlarr.db
```

**RPO (Recovery Point Objective):** Last snapshot (hourly/daily)
**RTO (Recovery Time Objective):** ~5 seconds

### PostgreSQL (Target)

```bash
# Approach 1: Logical backup (pg_dump)
pg_dump -Fc prowlarr_db > prowlarr_db.dump

# Approach 2: Physical backup (pg_basebackup)
pg_basebackup -D /backup/postgres -Ft -z

# Approach 3: ZFS snapshot (still works!)
zfs snapshot tank/postgres/data@daily

# Approach 4: Continuous archiving (WAL)
# Archive WAL files every 5 minutes
archive_command = 'cp %p /backup/wal_archive/%f'
```

**Advanced: Point-in-Time Recovery**
```bash
# Restore base backup
pg_basebackup restore

# Replay WAL until 15:30
recovery_target_time = '2024-01-01 15:30:00'

# Granularity: To the second
```

**RPO:** Seconds (with WAL archiving)
**RTO:** 5-30 minutes (restore + WAL replay)

---

## Performance Comparison

### Read Performance

| Operation | SQLite | PostgreSQL | Winner |
|-----------|--------|------------|--------|
| Single row by PK | 0.1ms | 0.2ms | SQLite (slightly) |
| Table scan (10k rows) | 50ms | 30ms | PostgreSQL |
| Complex JOIN | 100ms | 40ms | PostgreSQL |
| Concurrent reads | Limited | Excellent | PostgreSQL |

### Write Performance

| Operation | SQLite | PostgreSQL | Winner |
|-----------|--------|------------|--------|
| Single INSERT | 0.5ms | 0.8ms | SQLite (slightly) |
| Batch INSERT (1000 rows) | 500ms | 100ms | PostgreSQL |
| Concurrent writes | ❌ Serialized | ✅ Parallel | PostgreSQL |
| Write + Read | ❌ Blocks | ✅ No blocking | PostgreSQL |

**Conclusion:** PostgreSQL wins for any concurrent or complex workload.

---

## Decision Framework

### Stay with SQLite If:
- [x] Single application instance
- [x] Low write volume (<1000 writes/day)
- [x] No concurrent access needed
- [x] Simplicity is priority
- [x] Learning focus is on other areas

### Migrate to PostgreSQL If:
- [ ] Multiple applications sharing data
- [ ] High write volume or complex queries
- [ ] Need for replication/HA
- [ ] Learning database administration
- [ ] Planning to scale beyond single node
- [ ] Want advanced backup strategies (PITR)

**Your Context:**
Current state: SQLite is sufficient
Migration driver: Learning + future-proofing

---

## Related Notes

- [[Technical Architecture Overview]]
- [[ZFS Integration Patterns]]
- [[Industry Standard Comparisons]]
- [[Backup and Disaster Recovery]]

---

**References:**
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [SQLite Documentation](https://www.sqlite.org/docs.html)
- [PostgreSQL vs SQLite Comparison](https://www.sqlite.org/whentouse.html)
- [Migrating from SQLite to PostgreSQL](https://pgloader.readthedocs.io/en/latest/tutorial/tutorial.html#migrating-from-sqlite)
- [Servarr PostgreSQL Setup](https://wiki.servarr.com/postgres-setup)
