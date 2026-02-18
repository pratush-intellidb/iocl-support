# DC 2 Nodes + DR 2 Nodes – Detailed Steps (PostgreSQL 14, RHEL 9)

This document provides **detailed steps** for a 4-node layout:

- **DC (Data Center)**: 2 nodes – Primary + Streaming Standby (HA at DC).
- **DR (Disaster Recovery)**: 2 nodes – Primary (logical subscriber from DC) + Streaming Standby (HA at DR).

---

## Topology Overview

```
                    DC (Data Center)                    DR (Disaster Recovery)
                ┌─────────────────────────┐         ┌─────────────────────────┐
                │  DC-Primary (Node 1)    │         │  DR-Primary (Node 3)    │
                │  - Primary              │ logical │  - Subscriber (from DC)  │
                │  - Publisher            │ ──────► │  - Primary (for Node 4)   │
                └───────────┬─────────────┘         └───────────┬─────────────┘
                            │ streaming                         │ streaming
                            │ (physical)                        │ (physical)
                ┌───────────▼─────────────┐         ┌───────────▼─────────────┐
                │  DC-Standby (Node 2)    │         │  DR-Standby (Node 4)     │
                │  - Streaming replica    │         │  - Streaming replica     │
                └─────────────────────────┘         └─────────────────────────┘
```

**Summary of roles**

| Node       | Host role        | Replication role                                      |
|-----------|------------------|--------------------------------------------------------|
| DC-Primary  | Primary          | Publisher (logical) for DR-Primary                    |
| DC-Standby  | Streaming standby | Replica of DC-Primary (physical)                      |
| DR-Primary  | Primary          | Subscriber (logical) from DC-Primary; primary for Node 4 |
| DR-Standby  | Streaming standby | Replica of DR-Primary (physical)                      |

**IP/hostname assumptions (replace with your values)**

- DC-Primary:  `dc-primary.example.com` (e.g. 10.1.1.11)
- DC-Standby: `dc-standby.example.com` (e.g. 10.1.1.12)
- DR-Primary: `dr-primary.example.com` (e.g. 10.2.1.11)
- DR-Standby: `dr-standby.example.com` (e.g. 10.2.1.12)

---

## Phase 1: DC – Two Nodes (Primary + Streaming Standby)

### Step 1.1 – DC-Primary: Base PostgreSQL configuration

1. Install PostgreSQL 14 on RHEL 9 (both DC nodes).

2. On **DC-Primary**, set in `postgresql.conf`:

```conf
listen_addresses = '*'
port = 5432
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
```

3. In `pg_hba.conf` on **DC-Primary**, allow replication from DC-Standby and later from DR (for logical replication user):

```conf
# Replication from DC-Standby (streaming)
host  replication  replicator  10.1.1.12/32   scram-sha-256
# Replication from DR-Primary (logical – use same or dedicated user)
host  all          logical_repl  10.2.1.11/32   scram-sha-256
host  all          logical_repl  10.2.1.0/24    scram-sha-256
```

4. Create streaming replication user on **DC-Primary**:

```sql
CREATE USER replicator WITH REPLICATION PASSWORD 'your_replicator_password';
```

5. Restart DC-Primary:

```bash
sudo systemctl restart postgresql-14
```

---

### Step 1.2 – DC-Standby: Physical streaming replica

1. On **DC-Standby**, stop PostgreSQL if running, clear data directory (if this is a fresh clone):

```bash
sudo systemctl stop postgresql-14
sudo -u postgres rm -rf /var/lib/pgsql/14/data/*
```

2. Base backup from DC-Primary:

```bash
sudo -u postgres pg_basebackup -h dc-primary.example.com -D /var/lib/pgsql/14/data -U replicator -P -v -R -X stream
```

3. Start DC-Standby:

```bash
sudo systemctl start postgresql-14
```

4. Verify on DC-Primary:

```sql
SELECT application_name, state, sync_state, client_addr
FROM pg_stat_replication;
```

You should see DC-Standby in `streaming` state.

---

### Step 1.3 – DC-Primary: Enable logical replication (for DR)

1. On **DC-Primary**, set in `postgresql.conf`:

```conf
wal_level = logical
```

2. Restart DC-Primary:

```bash
sudo systemctl restart postgresql-14
```

3. Create user for logical replication (used by DR-Primary):

```sql
CREATE USER logical_repl WITH REPLICATION PASSWORD 'your_logical_repl_password';
GRANT USAGE ON SCHEMA public TO logical_repl;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO logical_repl;
```

4. Create publication (example: all tables in `public`; adjust as needed):

```sql
\c your_database_name
CREATE PUBLICATION dc_to_dr_pub FOR ALL TABLES IN SCHEMA public;
-- Or for specific tables: FOR TABLE t1, t2, t3;
```

5. Reload pg_hba if you added lines without restart:

```bash
sudo systemctl reload postgresql-14
```

---

## Phase 2: DR – Two Nodes (DR-Primary + DR-Standby)

### Step 2.1 – DR-Primary: Install and prepare (no subscription yet)

1. Install PostgreSQL 14 on both DR nodes (RHEL 9).

2. On **DR-Primary**, configure for future streaming to DR-Standby and for receiving logical replication:

Edit `postgresql.conf`:

```conf
listen_addresses = '*'
port = 5432
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
```

3. Create the **same database** and **same schema** (tables) as on DC-Primary. Example using schema-only dump from DC:

On a host that can reach DC-Primary:

```bash
pg_dump -h dc-primary.example.com -U postgres -s -d your_database_name > schema_only.sql
```

Copy `schema_only.sql` to DR-Primary, then:

```bash
sudo -u postgres psql -d your_database_name -f schema_only.sql
```

4. In `pg_hba.conf` on **DR-Primary**, allow DR-Standby and application/users as needed:

```conf
host  replication  replicator  10.2.1.12/32  scram-sha-256
host  all          all         10.2.1.0/24   scram-sha-256
```

5. Create streaming replication user (for DR-Standby):

```sql
CREATE USER replicator WITH REPLICATION PASSWORD 'your_dr_replicator_password';
```

6. Start DR-Primary:

```bash
sudo systemctl start postgresql-14
```

---

### Step 2.2 – DR-Primary: Create subscription to DC-Primary

1. On **DR-Primary**, create the logical subscription:

```sql
\c your_database_name

CREATE SUBSCRIPTION dc_to_dr_sub
CONNECTION 'host=dc-primary.example.com port=5432 dbname=your_database_name user=logical_repl password=your_logical_repl_password'
PUBLICATION dc_to_dr_pub
WITH (copy_data = true);
```

2. Verify subscription and replication slot on DC-Primary:

On **DC-Primary**:

```sql
SELECT slot_name, plugin, slot_type, active, restart_lsn FROM pg_replication_slots;
```

On **DR-Primary**:

```sql
SELECT * FROM pg_stat_subscription;
```

3. Wait for initial copy to finish (for large DBs, check logs and row counts). Then verify data:

```sql
-- Compare row counts on DC-Primary and DR-Primary for key tables
SELECT relname, n_live_tup FROM pg_stat_user_tables WHERE schemaname = 'public' ORDER BY relname;
```

---

### Step 2.3 – DR-Standby: Physical streaming replica of DR-Primary

1. On **DR-Standby**, stop PostgreSQL and clear data (fresh clone):

```bash
sudo systemctl stop postgresql-14
sudo -u postgres rm -rf /var/lib/pgsql/14/data/*
```

2. Base backup from DR-Primary:

```bash
sudo -u postgres pg_basebackup -h dr-primary.example.com -D /var/lib/pgsql/14/data -U replicator -P -v -R -X stream
```

3. Start DR-Standby:

```bash
sudo systemctl start postgresql-14
```

4. On **DR-Primary**, verify:

```sql
SELECT application_name, state, sync_state, client_addr
FROM pg_stat_replication;
```

You should see DR-Standby in `streaming` state.

---

## Phase 3: End-to-End Verification

### 3.1 Replication flow checks

| Check | Where to run | What to verify |
|-------|----------------|-----------------|
| DC streaming | DC-Primary | `pg_stat_replication`: DC-Standby streaming |
| Logical slot | DC-Primary | `pg_replication_slots`: slot for `dc_to_dr_sub`, active = t |
| Subscription | DR-Primary | `pg_stat_subscription`: worker running, no errors |
| DR streaming | DR-Primary | `pg_stat_replication`: DR-Standby streaming |

### 3.2 Test data flow DC → DR

1. On **DC-Primary**, insert/update/delete in a published table.
2. On **DR-Primary**, confirm the change appears.
3. On **DR-Standby**, confirm the same (via streaming from DR-Primary).

### 3.3 Firewall (RHEL 9)

On all four nodes, ensure PostgreSQL port is allowed:

```bash
sudo firewall-cmd --permanent --add-service=postgresql
sudo firewall-cmd --reload
```

---

## Phase 4: Failover / DR Promotion (High Level)

- **DC failure**: Promote DC-Standby to primary; reconfigure apps. Then either:
  - Point DR-Primary’s subscription to the new DC primary (recreate subscription or change connection), or
  - Keep DR as standalone until DC is restored.
- **DR-Primary failure**: Promote DR-Standby to primary. The new DR primary will need a new subscription to DC (or promoted DC primary) because subscription state is per-instance.

---

## Summary Checklist

- [ ] DC-Primary: `wal_level = logical`, publication created, `logical_repl` user and pg_hba.
- [ ] DC-Standby: streaming replica of DC-Primary.
- [ ] DR-Primary: same DB/schema, subscription to DC-Primary publication.
- [ ] DR-Standby: streaming replica of DR-Primary.
- [ ] All four: firewall, replication users, and connectivity verified.
- [ ] Test: write on DC-Primary → see on DR-Primary and DR-Standby.

---

*Document version: 1.0 | PostgreSQL 14 | RHEL 9 | DC 2 nodes + DR 2 nodes*
