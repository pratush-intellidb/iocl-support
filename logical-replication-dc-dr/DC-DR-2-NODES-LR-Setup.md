# DC 2 Nodes + DR 2 Nodes – Detailed Steps (PostgreSQL 14, RHEL 9)

This document provides **elaborate, step-by-step** instructions for a 4-node layout using **logical replication** from DC to DR:

- **DC (Data Center)**: 2 nodes – Primary (publisher) + Streaming Standby (physical HA at DC).
- **DR (Disaster Recovery)**: 2 nodes – Primary (logical subscriber from DC + primary for DR standby) + Streaming Standby (physical HA at DR).

---

## Table of Contents

1. [Topology Overview](#1-topology-overview)
2. [Prerequisites](#2-prerequisites)
3. [RHEL 9 and PostgreSQL 14 Conventions](#3-rhel-9-and-postgresql-14-conventions)
4. [Phase 1: DC – Two Nodes (Primary + Streaming Standby)](#4-phase-1-dc--two-nodes-primary--streaming-standby)
5. [Phase 2: DR – Two Nodes (DR-Primary + DR-Standby)](#5-phase-2-dr--two-nodes-dr-primary--dr-standby)
6. [Phase 3: Verification and Monitoring](#6-phase-3-verification-and-monitoring)
7. [Phase 4: Failover and Subscription Re-pointing](#7-phase-4-failover-and-subscription-re-pointing)
8. [Troubleshooting](#8-troubleshooting)
9. [Summary Checklist](#9-summary-checklist)

---

## 1. Topology Overview

```
                    DC (Data Center)                    DR (Disaster Recovery)
                ┌─────────────────────────┐         ┌─────────────────────────┐
                │  DC-Primary (Node 1)     │         │  DR-Primary (Node 3)     │
                │  - Primary               │ logical │  - Subscriber (from DC)   │
                │  - Publisher             │ ──────►  │  - Primary (for Node 4)    │
                └───────────┬──────────────┘         └───────────┬──────────────┘
                            │ streaming                           │ streaming
                            │ (physical)                         │ (physical)
                ┌───────────▼──────────────┐         ┌───────────▼──────────────┐
                │  DC-Standby (Node 2)     │         │  DR-Standby (Node 4)      │
                │  - Streaming replica     │         │  - Streaming replica      │
                └──────────────────────────┘         └──────────────────────────┘
```

### 1.1 Role summary

| Node        | Host role         | Replication role                                                                 |
|------------|-------------------|-----------------------------------------------------------------------------------|
| DC-Primary | Primary           | Publisher (logical) for DR-Primary; primary for DC-Standby (physical streaming) |
| DC-Standby | Streaming standby | Physical replica of DC-Primary                                                   |
| DR-Primary | Primary           | Subscriber (logical) from DC-Primary; primary for DR-Standby (physical streaming) |
| DR-Standby | Streaming standby | Physical replica of DR-Primary                                                  |

### 1.2 Naming and addresses (replace with your values)

| Node        | Example hostname           | Example IP    |
|-------------|----------------------------|---------------|
| DC-Primary  | dc-primary.example.com     | 10.1.1.11     |
| DC-Standby  | dc-standby.example.com     | 10.1.1.12     |
| DR-Primary  | dr-primary.example.com     | 10.2.1.11     |
| DR-Standby  | dr-standby.example.com     | 10.2.1.12     |

Use these consistently in `pg_hba.conf`, connection strings, and `application_name`.

**Command and copy-paste:** All commands in this document are written so they can be copied and pasted as a single block or line. Do not add line breaks inside a `-c "..."` string. If a command fails, run it again as shown without modifying spaces or newlines. Passwords containing single quotes should be escaped (e.g. `\'`) or use a `.pgpass` file.

---

## 2. Prerequisites

### 2.1 Software and versions

- **RHEL 9** on all four nodes (or equivalent).
- **PostgreSQL 14** (same minor version recommended on all nodes).
- Install from PostgreSQL official Yum repository if needed:

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql14-server postgresql14-contrib
sudo postgresql-14-setup initdb
sudo systemctl enable postgresql-14
```

### 2.2 Network and firewall

- **TCP port 5432** (or your chosen port) must be reachable:
  - DC-Primary ↔ DC-Standby
  - DC-Primary ↔ DR-Primary (for logical replication)
  - DR-Primary ↔ DR-Standby
- On each node (RHEL 9 with firewalld):

```bash
sudo firewall-cmd --permanent --add-service=postgresql
sudo firewall-cmd --reload
```

### 2.3 Connectivity checks

From each node, ensure you can reach the others. Replace `<target_host>` with the other node’s IP or hostname:

```bash
psql -h <target_host> -p 5432 -U postgres -d postgres -c "SELECT 1;"
```

### 2.4 Logical replication requirements

**Database-wise vs table-wise**

- **Subscription = database-wise:** Each subscription connects to **one database** on the publisher (`dbname=...` in the CONNECTION string). To replicate more than one database, create **one subscription per database** on the subscriber (each in the matching database).
- **Publication = table/schema/database-wise:** Inside that database, you choose **what** to replicate by creating a publication:
  - **Specific tables:** `FOR TABLE t1, t2, t3`
  - **All tables in a schema:** `FOR ALL TABLES IN SCHEMA public`
  - **All tables in the database:** `FOR ALL TABLES`

So: one subscription per publisher database; within each database, the publication defines which tables (or whole schema/database) are replicated.

- **Publisher (DC-Primary):** `wal_level = logical`.
- **Published tables:** Must have a **primary key** or **replica identity** (unique non-null index). Tables without a primary key can use `REPLICA IDENTITY FULL` (more WAL and slower apply).
- **Subscriber (DR-Primary):** Same database name and **identical table definitions** (same column names, types, and primary key/unique constraints). Data is copied by the subscription’s initial sync or pre-seeded manually.
- **Sequences:** Logical replication does not update sequence values on the subscriber. After initial sync, consider setting sequence values on the subscriber to match the publisher to avoid conflicts if applications insert on the subscriber (see Section 5.1 and Section 8.6).

---

## 3. RHEL 9 and PostgreSQL 14 Conventions

- **Data directory:** `/var/lib/pgsql/14/data`
- **Config files:** `postgresql.conf`, `pg_hba.conf` inside the data directory
- **Service name:** `postgresql-14`
- **Service commands:**

```bash
sudo systemctl status postgresql-14
sudo systemctl start postgresql-14
sudo systemctl stop postgresql-14
sudo systemctl restart postgresql-14
sudo systemctl reload postgresql-14
```

- **Logs (journalctl):**

```bash
sudo journalctl -u postgresql-14 -f
```

- **Edit config as postgres user:**

```bash
sudo -u postgres vi /var/lib/pgsql/14/data/postgresql.conf
sudo -u postgres vi /var/lib/pgsql/14/data/pg_hba.conf
```

---

## 4. Phase 1: DC – Two Nodes (Primary + Streaming Standby)

### 4.1 DC-Primary: Base configuration (physical streaming first)

1. **Locate config file** (run on DC-Primary):

```bash
sudo -u postgres psql -d postgres -t -c "SHOW config_file;"
```

2. **Edit `postgresql.conf`** (e.g. `/var/lib/pgsql/14/data/postgresql.conf`). Set or add:

```conf
listen_addresses = '*'
port = 5432
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
synchronous_standby_names = 'dc_standby_app'
```

- `synchronous_standby_names`: Use the `application_name` that DC-Standby will use in `primary_conninfo`. Commits on the primary will wait for this standby (depending on `synchronous_commit`).

3. **Edit `pg_hba.conf`.** Add lines so DC-Standby and (later) DR-Primary can connect. Use your actual IPs or subnets:

```conf
# IPv4: DC-Standby (streaming replication)
host    replication    replicator    10.1.1.12/32    scram-sha-256
# IPv4: DR-Primary (logical replication – connects as normal user to specific DB)
host    all            logical_repl  10.2.1.11/32    scram-sha-256
host    all            logical_repl  10.2.1.0/24     scram-sha-256
```

4. **Create streaming replication user** (for DC-Standby):

```bash
sudo -u postgres psql -d postgres -c "CREATE USER replicator WITH REPLICATION PASSWORD 'your_secure_replicator_password';"
```

5. **Restart DC-Primary:**

```bash
sudo systemctl restart postgresql-14
```

6. **Verify:**

```bash
sudo -u postgres psql -d postgres -c "SHOW wal_level;"
sudo -u postgres psql -d postgres -c "SHOW max_replication_slots;"
```

---

### 4.2 DC-Standby: Physical streaming replica

1. **Stop PostgreSQL and clear data** (only for a fresh clone; skip if reusing an existing standby):

```bash
sudo systemctl stop postgresql-14
sudo -u postgres rm -rf /var/lib/pgsql/14/data/*
```

2. **Take base backup from DC-Primary.** Replace `dc-primary.example.com` and the replicator password:

```bash
sudo -u postgres pg_basebackup -h dc-primary.example.com -p 5432 -U replicator -D /var/lib/pgsql/14/data -Fp -X stream -R -P -v
```

- **-Fp:** Plain format (files as-is).
- **-X stream** (or **-Xs**): Stream WAL while backup runs (requires a replication connection).
- **-R:** Create `standby.signal` and `primary_conninfo` so the instance starts as a standby.
- **-P:** Show progress.

3. **Set application_name for sync standby.** The `-R` option in PostgreSQL 14 writes `primary_conninfo` into `postgresql.auto.conf`; it may not include `application_name`. You must set the same name as in `synchronous_standby_names` on the primary. Edit:

```bash
sudo -u postgres vi /var/lib/pgsql/14/data/postgresql.auto.conf
```

Ensure the line includes `application_name=dc_standby_app` (add it if missing):

```conf
primary_conninfo = 'host=dc-primary.example.com port=5432 user=replicator password=your_secure_replicator_password application_name=dc_standby_app'
```

4. **Start DC-Standby:**

```bash
sudo systemctl start postgresql-14
```

5. **On DC-Primary, verify streaming:**

```bash
sudo -u postgres psql -d postgres -c "SELECT application_name, state, sync_state, client_addr FROM pg_stat_replication;"
```

You should see one row with `application_name = dc_standby_app`, `state = streaming`, and `sync_state = sync` (if you have one sync standby and `synchronous_commit` requires it).

---

### 4.3 DC-Primary: Enable logical replication (for DR)

1. **Edit `postgresql.conf`** and set:

```conf
wal_level = logical
```

2. **Restart DC-Primary** (required when changing `wal_level`):

```bash
sudo systemctl restart postgresql-14
```

3. **Verify:**

```bash
sudo -u postgres psql -d postgres -c "SHOW wal_level;"
```

Expected: `logical`.

4. **Create user for logical replication** (used by DR-Primary to connect and fetch changes). `CREATE USER` is cluster-wide (run once per cluster); the `GRANT` and publication steps are per-database. Run each statement separately (database context for CREATE USER does not matter; for multiple databases, create `logical_repl` once, then GRANT and create publications in each database):

```bash
sudo -u postgres psql -d your_database_name -c "CREATE USER logical_repl WITH REPLICATION PASSWORD 'your_secure_logical_repl_password';"
sudo -u postgres psql -d your_database_name -c "GRANT USAGE ON SCHEMA public TO logical_repl;"
sudo -u postgres psql -d your_database_name -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;"
sudo -u postgres psql -d your_database_name -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO logical_repl;"
```

If you publish tables in multiple schemas, repeat `GRANT USAGE` and `GRANT SELECT` for those schemas.

5. **Create publication.** Run **exactly one** of the following (choose A, B, or C):

**Option A – All tables in schema public:**
```bash
sudo -u postgres psql -d your_database_name -c "CREATE PUBLICATION dc_to_dr_pub FOR ALL TABLES IN SCHEMA public;"
```

**Option B – Specific tables only:**
```bash
sudo -u postgres psql -d your_database_name -c "CREATE PUBLICATION dc_to_dr_pub FOR TABLE t1, t2, t3;"
```

**Option C – All tables in database:**
```bash
sudo -u postgres psql -d your_database_name -c "CREATE PUBLICATION dc_to_dr_pub FOR ALL TABLES;"
```

6. **List publications:**

```bash
sudo -u postgres psql -d your_database_name -c "\dRp+"
```

7. **Reload pg_hba** if you added entries after the last restart:

```bash
sudo systemctl reload postgresql-14
```

---

## 5. Phase 2: DR – Two Nodes (DR-Primary + DR-Standby)

### 5.1 DR-Primary: Install and initial configuration

1. **Install PostgreSQL 14** on both DR nodes (see Prerequisites). Ensure the service is running (e.g. `sudo systemctl start postgresql-14`) so you can create databases and run commands.

2. **Create the same database** as on DC (same name):

```bash
sudo -u postgres psql -d postgres -c "CREATE DATABASE your_database_name;"
```

3. **Copy schema from DC-Primary (no data).** From a host that can reach DC-Primary:

```bash
pg_dump -h dc-primary.example.com -p 5432 -U postgres -s -d your_database_name -n public -f schema_only.sql
```

- **-s:** Schema only (no data).
- **-n public:** Schema to dump; repeat for other schemas if needed (e.g. `-n public -n other_schema`).
- If the database uses **extensions** (e.g. `uuid-ossp`, `pg_trgm`), the dump may include `CREATE EXTENSION`. Install the same extensions on DR-Primary before restoring. Extension names with a hyphen must be double-quoted in SQL. Examples:
  - `sudo -u postgres psql -d your_database_name -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"`
  - `sudo -u postgres psql -d your_database_name -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"`
  (Repeat for each extension used by your schema.)

Copy `schema_only.sql` to DR-Primary. From the directory containing the file (or use the full path), run:

```bash
sudo -u postgres psql -d your_database_name -f schema_only.sql
```

**Important:** Table definitions (column names, types, primary keys) must match the publisher. If you add tables later to the publication, create them on the subscriber first, then add them to the publication and run `ALTER SUBSCRIPTION ... REFRESH PUBLICATION` on the subscriber.

**Sequences:** The schema-only dump normally includes `CREATE SEQUENCE` for sequences used by tables (e.g. SERIAL columns), so sequence **objects** are created when you restore `schema_only.sql`. To confirm sequences exist after restore, run: `SELECT sequencename FROM pg_sequences WHERE schemaname = 'public';` If a sequence is missing, create it with the same definition as on the publisher (e.g. from the dump or `\d sequence_name` on the publisher), then set its value as below. Logical replication does **not** update sequence current values, so after the initial subscription copy you must set each sequence’s value on the subscriber to match the publisher (otherwise inserts on the subscriber can cause duplicate key errors).

**Step 1 – List sequences** (optional; run on publisher or subscriber to see sequence names). This uses the `pg_sequences` catalog (PostgreSQL 10+):

```bash
sudo -u postgres psql -d your_database_name -t -c "SELECT schemaname || '.' || sequencename FROM pg_sequences WHERE schemaname = 'public';"
```

**Step 2 – On publisher**, get the current value for each sequence. Replace `myseq` with your sequence name (e.g. `users_id_seq`). Use `pg_sequences` so the command works for any sequence name, including those with hyphens:

```bash
sudo -u postgres psql -d your_database_name -t -A -c "SELECT last_value FROM pg_sequences WHERE schemaname = 'public' AND sequencename = 'myseq';"
```

Note the number printed (e.g. `12345`). If the result is empty, the sequence name may be wrong or the sequence may be in another schema (change `schemaname` in the WHERE clause).

**Step 3 – On subscriber**, set the same value. Use the **actual number** from Step 2 (e.g. `12345`). Replace `myseq` with your sequence name (same as in Step 2) and `12345` with the value from the publisher.

For a **standard sequence name** (letters, numbers, underscores only, e.g. `users_id_seq`):

```bash
sudo -u postgres psql -d your_database_name -c "SELECT setval('public.myseq'::regclass, 12345);"
```

For a **sequence name with hyphens or mixed case**, the identifier must be double-quoted in SQL. From the shell, escape the inner double quotes with a backslash:

```bash
sudo -u postgres psql -d your_database_name -c "SELECT setval('\"public\".\"my-seq\"'::regclass, 12345);"
```

(Replace `my-seq` with your sequence name and `12345` with the value from Step 2.)

Repeat Step 2 and Step 3 for each sequence used by published tables.

---

**Automated – sync all sequences (many sequences)**

When there are many sequences, use the following to generate a script on the publisher and run it on the subscriber. Replace `your_database_name` and paths as needed.

**A. On the publisher** (or any host that can connect to the publisher): generate a SQL file containing one `setval` per sequence, using the publisher’s current `last_value`. This works for any sequence name (including hyphens and mixed case). Run **one** of the following (first: all user schemas; second: schema `public` only).

All user schemas (excludes `pg_catalog`):

```bash
sudo -u postgres psql -d your_database_name -t -A -o /tmp/setval_all_sequences.sql -c "SELECT 'SELECT setval(' || quote_literal(quote_ident(schemaname) || '.' || quote_ident(sequencename)) || '::regclass, ' || last_value || ');' FROM pg_sequences WHERE schemaname NOT IN ('pg_catalog') ORDER BY schemaname, sequencename;"
```

Schema `public` only:

```bash
sudo -u postgres psql -d your_database_name -t -A -o /tmp/setval_all_sequences.sql -c "SELECT 'SELECT setval(' || quote_literal(quote_ident(schemaname) || '.' || quote_ident(sequencename)) || '::regclass, ' || last_value || ');' FROM pg_sequences WHERE schemaname = 'public' ORDER BY sequencename;"
```

**B. Copy the file to the subscriber** (if the publisher and subscriber are different hosts). Replace `subscriber_host` with the DR-Primary hostname or IP:

```bash
scp /tmp/setval_all_sequences.sql subscriber_host:/tmp/
```

**C. On the subscriber**, run the generated SQL:

```bash
sudo -u postgres psql -d your_database_name -f /tmp/setval_all_sequences.sql
```

All sequences listed in `pg_sequences` (for the chosen schema(s)) will be set to the publisher’s `last_value`. You can inspect the file before running: `cat /tmp/setval_all_sequences.sql`.

**One-shot pipeline (from a host that can reach both publisher and subscriber):** Generate from the publisher and run on the subscriber without a temp file. Replace `dc-primary.example.com` and `dr-primary.example.com` with your hostnames (and add `-p 5432` to each `psql` if not default):

```bash
sudo -u postgres psql -h dc-primary.example.com -U postgres -d your_database_name -t -A -c "SELECT 'SELECT setval(' || quote_literal(quote_ident(schemaname) || '.' || quote_ident(sequencename)) || '::regclass, ' || last_value || ');' FROM pg_sequences WHERE schemaname = 'public' ORDER BY sequencename;" | sudo -u postgres psql -h dr-primary.example.com -U postgres -d your_database_name -f -
```

First `psql` connects to the **publisher** and outputs the `setval` statements; the second runs them on the **subscriber**. Ensure `pg_hba.conf` on both allows the connection. For non-interactive use (e.g. cron), use a `.pgpass` file or `PGPASSWORD` so neither `psql` prompts for a password.

---

**When new sequences are added later**

Logical replication does **not** replicate sequence values. When you add new tables (with SERIAL/IDENTITY columns) or new standalone sequences on the publisher and add them to the publication, the subscriber will have the sequence **objects** (if you applied the schema there), but their **current values** will not match the publisher. Inserts on the subscriber that use those sequences can then cause duplicate key errors or gaps. You must re-sync sequence values whenever new sequences (or new tables that use sequences) are introduced.

**Option 1 – Re-run the full automated sync (recommended)**

The simplest and safest approach is to run the same **Automated – sync all sequences** steps again (A, B, C above) or the **one-shot pipeline**. That script uses the publisher’s current `pg_sequences.last_value` for every sequence, so it will:

- Update all existing sequences to the publisher’s current value.
- Set values for any **new** sequences that have appeared since the last sync.

No need to identify which sequences are new. Run it:

- **After** you have applied the new schema on the subscriber (new table or new sequence created).
- **After** the new table/sequence is included in the publication and (if applicable) initial copy or refresh is done.

Repeat the same commands as in the initial setup: either generate `/tmp/setval_all_sequences.sql` on the publisher (with the same schema filter you use elsewhere, e.g. `public` only or all user schemas), copy it to the subscriber, and run it on the subscriber; or use the one-shot pipeline from a host that can reach both.

**Option 2 – Sync only the new sequence(s)**

Use this when you want to touch only the sequences that were added recently.

1. **Find new sequences on the publisher** (e.g. compare to a previous list, or list by creation time). For example, list sequences in `public`:
   ```bash
   sudo -u postgres psql -d your_database_name -t -c "SELECT schemaname || '.' || sequencename FROM pg_sequences WHERE schemaname = 'public' ORDER BY sequencename;"
   ```
2. **Ensure the sequence exists on the subscriber.** If the new sequence was created by a schema change (e.g. new table with SERIAL), restore or apply that schema on the subscriber first so the sequence object exists. For a standalone sequence, create it on the subscriber with the same definition as on the publisher (e.g. from a schema dump or `\d sequence_name` on the publisher).
3. **For each new sequence**, run **Step 2** and **Step 3** from the manual sequence sync above: on the publisher get `last_value` from `pg_sequences`, then on the subscriber run `setval(..., value)` with that number. Alternatively, generate a setval script only for a subset of sequences (e.g. by filtering in the `SELECT ... FROM pg_sequences` with a specific schema or naming pattern), copy it to the subscriber, and run it.

**When to run**

- After adding a new table (or new columns that use sequences) to the publication and applying the same schema on the subscriber.
- After creating a new standalone sequence and adding it to the publication (if applicable).
- Optionally on a schedule (e.g. daily cron) if you frequently add tables/sequences and want the subscriber’s sequence values to stay close to the publisher. If you run it while writes are ongoing, the subscriber may briefly be slightly behind the publisher until the next run; for most use cases this is acceptable.

**Scheduled (cron) re-sync (optional)**

To re-run the full sequence sync periodically (e.g. daily), use the one-shot pipeline in a script and call it from cron. Use `.pgpass` or `PGPASSWORD` so `psql` does not prompt. Example (run as a user that can `sudo -u postgres` or as postgres):

```bash
# Example: sync sequences for database mydb, daily at 02:00 (schema public only)
0 2 * * * sudo -u postgres psql -h dc-primary.example.com -U postgres -d mydb -t -A -c "SELECT 'SELECT setval(' || quote_literal(quote_ident(schemaname) || '.' || quote_ident(sequencename)) || '::regclass, ' || last_value || ');' FROM pg_sequences WHERE schemaname = 'public' ORDER BY sequencename;" | sudo -u postgres psql -h dr-primary.example.com -U postgres -d mydb -f -
```

Replace `mydb`, `dc-primary.example.com`, and `dr-primary.example.com` with your database and hostnames. This keeps all sequences (including newly added ones) aligned with the publisher without having to re-run the full manual procedure each time.

---

4. **On DR-Primary**, edit `postgresql.conf` (e.g. `/var/lib/pgsql/14/data/postgresql.conf`):

```conf
listen_addresses = '*'
port = 5432
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
synchronous_standby_names = 'dr_standby_app'
```

5. **On DR-Primary**, edit `pg_hba.conf` (allow DR-Standby and, if needed, application hosts):

```conf
host  replication  replicator  10.2.1.12/32  scram-sha-256
host  all          all         10.2.1.0/24   scram-sha-256
```

6. **Create streaming replication user** (for DR-Standby):

```bash
sudo -u postgres psql -d postgres -c "CREATE USER replicator WITH REPLICATION PASSWORD 'your_dr_replicator_password';"
```

7. **Start or restart DR-Primary** so that `postgresql.conf` and `pg_hba.conf` changes take effect:

```bash
sudo systemctl restart postgresql-14
```

(If this is a fresh install and the service was not started yet, use `start` instead of `restart`.)

---

### 5.2 DR-Primary: Create subscription to DC-Primary

1. **Create the subscription.** Replace host, dbname, user, and password with your values. The CONNECTION string must be a single line (no line breaks). If the password contains single quotes, escape them with a backslash (e.g. `\'`) or use a `.pgpass` file. For production, prefer `.pgpass` instead of embedding the password.

```bash
sudo -u postgres psql -d your_database_name -c "CREATE SUBSCRIPTION dc_to_dr_sub CONNECTION 'host=dc-primary.example.com port=5432 dbname=your_database_name user=logical_repl password=your_secure_logical_repl_password' PUBLICATION dc_to_dr_pub WITH (copy_data = true, enabled = true);"
```

- **copy_data = true:** Initial table sync from publisher to subscriber (recommended for new setup).
- **enabled = true:** Subscription is active immediately (default; can be omitted).

2. **Verify on DC-Primary – replication slot created and active:**

```bash
sudo -u postgres psql -d your_database_name -c "SELECT slot_name, plugin, slot_type, active, restart_lsn, confirmed_flush_lsn FROM pg_replication_slots;"
```

You should see a slot named after the subscription (e.g. `dc_to_dr_sub`) with `active = t`.

3. **Verify on DR-Primary – subscription state:**

```bash
sudo -u postgres psql -d your_database_name -c "SELECT subname, subenabled, subconninfo FROM pg_subscription;"
sudo -u postgres psql -d your_database_name -c "SELECT * FROM pg_stat_subscription;"
```

4. **Wait for initial data copy** (for large databases this can take a long time). Monitor on DR-Primary:

```bash
sudo journalctl -u postgresql-14 -f
```

You can also compare row counts between DC-Primary and DR-Primary (see Phase 3).

---

### 5.3 DR-Standby: Physical streaming replica of DR-Primary

1. **Stop PostgreSQL and clear data** on DR-Standby (fresh clone only):

```bash
sudo systemctl stop postgresql-14
sudo -u postgres rm -rf /var/lib/pgsql/14/data/*
```

2. **Base backup from DR-Primary:**

```bash
sudo -u postgres pg_basebackup -h dr-primary.example.com -p 5432 -U replicator -D /var/lib/pgsql/14/data -Fp -X stream -R -P -v
```

3. **Set application_name** for sync standby. Edit `postgresql.auto.conf` (or the generated standby config):

```conf
primary_conninfo = 'host=dr-primary.example.com port=5432 user=replicator password=your_dr_replicator_password application_name=dr_standby_app'
```

4. **Start DR-Standby:**

```bash
sudo systemctl start postgresql-14
```

5. **On DR-Primary, verify streaming:**

```bash
sudo -u postgres psql -d postgres -c "SELECT application_name, state, sync_state, client_addr FROM pg_stat_replication;"
```

Expect one row with `application_name = dr_standby_app` and `state = streaming`.

---

## 6. Phase 3: Verification and Monitoring

### 6.1 Replication flow summary

| What to check        | Where to run   | Command / what to see |
|----------------------|----------------|------------------------|
| DC physical streaming| DC-Primary     | `SELECT * FROM pg_stat_replication;` – DC-Standby streaming |
| Logical slot         | DC-Primary     | `SELECT * FROM pg_replication_slots;` – slot for subscription, `active = t` |
| Subscription         | DR-Primary     | `SELECT * FROM pg_stat_subscription;` – no errors |
| DR physical streaming| DR-Primary     | `SELECT * FROM pg_stat_replication;` – DR-Standby streaming |

### 6.2 Compare row counts (DC vs DR)

Run on both DC-Primary and DR-Primary (same database). In interactive psql (e.g. `psql -d your_database_name`) or as a single-line `-c`:

```bash
sudo -u postgres psql -d your_database_name -c "SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables WHERE schemaname = 'public' ORDER BY schemaname, relname;"
```

Row counts should match for published tables after initial sync and under normal replication.

### 6.3 Test data flow

1. On **DC-Primary**, in a published table (replace `your_table` and column names with a real published table):

```bash
sudo -u postgres psql -d your_database_name -c "INSERT INTO your_table (id, data) VALUES (999, 'test from DC');"
```

2. On **DR-Primary**, confirm the row exists (e.g. `SELECT * FROM your_table WHERE id = 999;`).
3. On **DR-Standby**, confirm the same row (replicated via physical streaming from DR-Primary).

### 6.4 Useful monitoring queries

**On DC-Primary – logical slot lag (bytes behind current WAL):**

```bash
sudo -u postgres psql -d your_database_name -c "SELECT slot_name, active, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes FROM pg_replication_slots WHERE slot_type = 'logical';"
```

**On DR-Primary – subscription worker:**

```bash
sudo -u postgres psql -d your_database_name -c "SELECT pid, received_lsn, last_msg_send_time, last_msg_receipt_time FROM pg_stat_subscription;"
```

---

## 7. Phase 4: Failover and Subscription Re-pointing

### 7.1 DC failover (promote DC-Standby to primary)

1. On **DC-Standby**, promote to primary. Either run from the shell:

```bash
sudo -u postgres pg_ctl promote -D /var/lib/pgsql/14/data
```

Or from **psql on DC-Standby** — run this **on the DC-Standby server** (not on the primary), connecting to any database:

```bash
sudo -u postgres psql -d postgres -c "SELECT pg_promote();"
```

2. Reconfigure applications to use the new DC primary (former DC-Standby).

3. **DR subscription:** The subscription on DR-Primary still points to the old DC-Primary (now down). To continue logical replication from the new DC primary (former DC-Standby):

   **Option A – Alter connection (recommended in PostgreSQL 14):** Change the subscription to point to the new DC primary host without dropping it. Run on DR-Primary (single line):

```bash
sudo -u postgres psql -d your_database_name -c "ALTER SUBSCRIPTION dc_to_dr_sub CONNECTION 'host=new-dc-primary.example.com port=5432 dbname=your_database_name user=logical_repl password=your_secure_logical_repl_password';"
```

   **Option B – Drop and recreate:** If you prefer to recreate the subscription, run on DR-Primary:

```bash
sudo -u postgres psql -d your_database_name -c "DROP SUBSCRIPTION dc_to_dr_sub;"
sudo -u postgres psql -d your_database_name -c "CREATE SUBSCRIPTION dc_to_dr_sub CONNECTION 'host=new-dc-primary.example.com port=5432 dbname=your_database_name user=logical_repl password=xxx' PUBLICATION dc_to_dr_pub WITH (copy_data = false);"
```

**Before failover:** Ensure **both** DC-Primary and DC-Standby have `pg_hba.conf` entries allowing `logical_repl` from DR-Primary’s IP, so that after promotion the new DC primary already accepts the connection. The new DC primary (former standby) already has `wal_level = logical`, the publication, and the `logical_repl` user (replicated from the old primary).

### 7.2 DR failover (promote DR-Standby to primary)

1. On **DR-Standby**, promote to primary (same commands as in Section 7.1, using DR-Standby’s data directory).
2. Applications at DR should point to the new DR primary.
3. **Subscription:** The subscription existed only on the old DR-Primary. The new DR primary (former DR-Standby) has the same data (from streaming) but **no subscription**. To resume logical replication from DC:
   - The database and schema are already present on the new DR primary (they were streamed from the old DR-Primary).
   - On the **publisher (DC-Primary)**, drop the old replication slot so a new subscription can use the same slot name (the old slot is orphaned because the old DR-Primary is gone):
     ```bash
     sudo -u postgres psql -d your_database_name -c "SELECT pg_drop_replication_slot('dc_to_dr_sub');"
     ```
   - On the **new DR primary**, create the subscription with `copy_data = false` (data is already there from physical replication):
     ```bash
     sudo -u postgres psql -d your_database_name -c "CREATE SUBSCRIPTION dc_to_dr_sub CONNECTION 'host=dc-primary.example.com port=5432 dbname=your_database_name user=logical_repl password=your_secure_logical_repl_password' PUBLICATION dc_to_dr_pub WITH (copy_data = false);"
     ```
     Replace the host with the current DC primary (e.g. after DC failover, use the new DC primary hostname). Ensure `pg_hba.conf` on the DC primary allows `logical_repl` from the new DR primary’s IP.

---

## 8. Troubleshooting

### 8.1 Logical replication slot inactive

- **Symptom:** On DC-Primary, `pg_replication_slots` shows `active = f` for the subscription slot.
- **Causes:** DR-Primary down, network/firewall blocking, or wrong credentials in subscription connection string.
- **Actions:** Check DR-Primary is up; test connectivity from DR-Primary to DC-Primary (e.g. `psql -h dc-primary ...`); check `pg_stat_subscription` and DR-Primary logs for errors.

### 8.2 Connection refused or authentication failed

- **pg_hba.conf:** Ensure DC-Primary has a `host ... logical_repl ... scram-sha-256` line for DR-Primary’s IP (or subnet).
- **Password:** Ensure the password in the subscription connection string matches the `logical_repl` user on DC-Primary.
- **Reload:** After editing `pg_hba.conf`, run `sudo systemctl reload postgresql-14` on DC-Primary.
- **SELinux (RHEL 9):** If connections fail with permission denied despite pg_hba and firewall being correct, check SELinux. You may need to allow network connectivity for PostgreSQL (e.g. `setsebool -P postgresql_can_network_connect 1`). Verify with `getenforce` and audit logs.

### 8.3 Schema or table definition mismatch

- **Symptom:** Subscription worker errors about missing column or type mismatch.
- **Requirement:** Subscriber tables must have the same column names, types, and primary key (or replica identity) as the publisher.
- **Action:** Alter the subscriber table to match, or drop and recreate it from a schema dump. Then on the **subscriber (DR-Primary)** run one of the following (to copy data for the new/empty table, use `copy_data = true`; otherwise `false`):

```bash
sudo -u postgres psql -d your_database_name -c "ALTER SUBSCRIPTION dc_to_dr_sub REFRESH PUBLICATION WITH (copy_data = true);"
```

or

```bash
sudo -u postgres psql -d your_database_name -c "ALTER SUBSCRIPTION dc_to_dr_sub REFRESH PUBLICATION WITH (copy_data = false);"
```

### 8.4 Initial copy very slow or stuck

- Check DR-Primary logs for apply errors.
- On DC-Primary, ensure `logical_repl` has `SELECT` on all published tables and that no long-running transaction is holding back the snapshot.
- For very large tables, consider pre-seeding the subscriber (e.g. with `pg_dump`/`pg_restore`) and creating the subscription with `copy_data = false` for those tables (or for the whole subscription if everything is pre-seeded).

### 8.5 Replication slot holds too much WAL

- If the subscriber is down for a long time, the logical slot on the publisher can prevent WAL from being removed and fill disk.
- **Mitigation:** Bring the subscriber back up so it consumes WAL. If you must remove the slot: drop the subscription on the subscriber first (`DROP SUBSCRIPTION dc_to_dr_sub`), which normally drops the slot on the publisher. If the subscriber is unreachable, on the publisher you can force-drop the slot with `SELECT pg_drop_replication_slot('dc_to_dr_sub');` — only if you accept that this subscription cannot be resumed without resync.

### 8.6 Sequence out of sync (duplicate key on subscriber)

- **Symptom:** Inserts on the subscriber fail with duplicate key, or sequence-backed columns collide with replicated rows.
- **Cause:** Logical replication does not replicate sequence values; the subscriber’s sequences may be behind the publisher’s.
- **Action:** After initial sync (or after backfill), set each sequence on the subscriber to at least the maximum value used in the corresponding column. Run on the **subscriber**. Replace `public.myseq` with your sequence name, and `public.mytable` / `id` with the table and column that use this sequence (e.g. `id` for a primary key):
  ```bash
  sudo -u postgres psql -d your_database_name -c "SELECT setval('public.myseq'::regclass, (SELECT COALESCE(max(id), 1) FROM public.mytable));"
  ```
  `COALESCE(max(id), 1)` ensures that if the table is empty, the sequence is set to 1. Repeat for each sequence.
- **New sequences added later:** If you added new tables or sequences after the initial setup, re-sync sequence values using the same automated steps (Section 5.1 – *When new sequences are added later*): re-run the full setval script or one-shot pipeline, or sync only the new sequence(s) manually.

---

## 9. Summary Checklist

- [ ] **Prerequisites:** PostgreSQL 14 and RHEL 9 on all four nodes; firewall allows 5432; connectivity tested.
- [ ] **DC-Primary:** `wal_level = logical`, `replicator` and `logical_repl` users, pg_hba for DC-Standby and DR-Primary, publication created.
- [ ] **DC-Standby:** Base backup from DC-Primary with `-R`, `application_name` set, streaming and (if used) sync standby verified.
- [ ] **DR-Primary:** Same database and schema as DC; subscription to DC-Primary publication; initial sync completed; `replicator` and pg_hba for DR-Standby.
- [ ] **DR-Standby:** Base backup from DR-Primary with `-R`, `application_name` set, streaming verified.
- [ ] **Verification:** Row counts match; test insert on DC visible on DR-Primary and DR-Standby.
- [ ] **Sequences (if needed):** Sequence values on DR-Primary set to match publisher (manual steps or automated script), so subscriber inserts do not conflict.
- [ ] **After adding new tables/sequences:** Re-run sequence sync (Section 5.1 – *When new sequences are added later*) so new sequences have correct values on the subscriber.
- [ ] **Documentation:** Failover and subscription re-pointing steps documented and tested where possible.

---

*Document version: 2.2 | PostgreSQL 14 | RHEL 9 | DC 2 nodes + DR 2 nodes (logical replication)*
