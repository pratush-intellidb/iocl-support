## PostgreSQL 14 – `mis_newgen` Performance & Indexing Resolution

### 1. Executive Summary

- The `mis_newgen` view joins many tables and can return ~50 crore (~500M) rows.
- Running `SELECT * FROM mis_newgen;` on the production primary exhausts 56 GB RAM and the backend is killed.
- 3–4 base tables have been identified where additional indexes would make `mis_newgen` fast and safe, but the primary is a **write‑intensive** system and these indexes would slow inserts and increase WAL.
- The existing secondary is a **physical streaming replica (hot standby)** and is **read‑only** by design, so `CREATE INDEX` fails there.
- An additional empty server is available and can be used to host reporting workloads.

**Resolution (high level):**

- Keep the production primary **write‑optimized** (no new heavy indexes).
- Keep the existing secondary as a **HA standby** (unchanged, read‑only).
- Use the empty server as a **reporting replica** (data can be loaded via **logical replication** and/or **pg_basebackup**; see Section 5 for options), where we:
  - Add the required reporting indexes on the 3–4 base tables.
  - Create a materialized view `mis_newgen_mv` to replace `mis_newgen` for reporting.
  - Run all heavy `mis_newgen` queries without impacting the primary.

---

### 2. Root Cause Analysis

1. **Memory exhaustion on primary**
   - `SELECT * FROM mis_newgen;` attempts to materialize an extremely large result set.
   - This drives very high memory usage for joins, sorts, and buffering, eventually exhausting 56 GB RAM and causing the backend to be killed by the OS.

2. **Secondary is read‑only hot standby**
   - The current secondary is a **physical streaming replica** running in recovery (`pg_is_in_recovery() = true`, `transaction_read_only = on`).
   - In this mode, PostgreSQL allows only read queries and some `SET` commands; all DDL/DML (including `CREATE INDEX`, `CREATE TABLE`, `REFRESH MATERIALIZED VIEW`) are blocked by design.
   - As a result, this secondary cannot host additional reporting indexes or materialized views.

3. **Primary is a write‑heavy OLTP system**
   - Adding multiple large indexes on the primary’s base tables would:
     - Increase per‑insert CPU, I/O, and WAL.
     - Reduce maximum write throughput and risk slowing the application’s critical path.
   - Therefore, putting reporting indexes directly on the primary is not acceptable.

---

### 3. Immediate Operational Mitigation (Short Term)

Until the full solution is implemented:

- **Do not run `SELECT * FROM mis_newgen` on the primary.**
  - Only run **filtered, targeted queries** (e.g. by date, key ranges, specific columns).
  - Avoid ad‑hoc full‑scan queries on `mis_newgen`.
- Keep **conservative settings** on the primary:
  - Smaller `work_mem` and a reasonable `statement_timeout` for application users to prevent any single query from consuming all memory.

These actions reduce immediate risk while we put the dedicated reporting architecture in place.

---

### 4. Target Architecture (Final State)

#### 4.1. Roles of Each Server

1. **Primary (Production OLTP)**
   - Write‑intensive, with only the minimal indexes required for the application.
   - `wal_level = logical` to support logical replication.
   - Protected from heavy reporting and ad‑hoc analytics.

2. **Current Secondary (HA Standby)**
   - Stays as a **physical streaming replica** for high availability and failover.
   - Remains read‑only; no additional indexes created here.

3. **Empty Server (New Logical Reporting Replica)**
   - Configured as a **logical subscriber** to selected tables on the primary.
   - Normal read‑write PostgreSQL instance (not in recovery).
   - Hosts:
     - Extra reporting indexes on the identified base tables.
     - A materialized view that replaces `mis_newgen` for reporting.
     - All heavy analytical queries.

---

### 5. Reporting Server Setup: Options (Including pg_basebackup)

You can build the reporting server in three ways. Choose based on whether you prefer **logical replication only**, **pg_basebackup only** (periodic refresh), or **pg_basebackup to seed** then **logical replication** for ongoing sync.

| Option | How reporting server gets data | Freshness | Use when |
|--------|--------------------------------|-----------|----------|
| **A**  | Logical replication only (subscription with `copy_data = true`) | Near real-time | You want continuous sync and can wait for initial copy. |
| **B**  | pg_basebackup only (restore as standalone, no replication) | As of last backup (e.g. nightly) | You are okay with periodic refresh (e.g. daily) and can run backup/restore + index creation on a schedule. |
| **C**  | pg_basebackup to seed, then logical replication for ongoing changes | Near real-time after initial seed | Database is very large; you want a fast initial load, then continuous sync. |

#### Option A: Logical Replication Only (Default in this document)

- Primary has `wal_level = logical` and a **publication** for the `mis_newgen` base tables.
- On the **empty server** (normal instance, not standby): create same schemas/tables, then **CREATE SUBSCRIPTION** with `copy_data = true`.
- Initial copy and ongoing changes are both via logical replication. No pg_basebackup.
- **Pros:** Single mechanism, continuous sync, no file-level restore.  
- **Cons:** Initial copy can be slow for very large DBs.

#### Option B: pg_basebackup Only (Periodic Full Refresh)

- Use **pg_basebackup** to take a full backup from the primary.
- Restore that backup on the empty server and **start as a normal instance** (do **not** start as standby — no `standby.signal`, no `primary_conninfo` in recovery). So the server is standalone and read-write.
- On this copy: create the 3–4 reporting indexes and, if needed, the materialized view. Run all heavy `mis_newgen` queries here.
- Data is **as of the backup time**. To refresh:
  - Take a new pg_basebackup (e.g. nightly).
  - Stop the reporting instance, replace its data directory with the new backup (or restore to a new directory and swap).
  - Start the instance and **re-create the reporting indexes** (and MV) after each restore, because the restored data is a copy of the primary, which has no reporting indexes.
- **Pros:** Uses only tools you may already use (pg_basebackup); no logical replication setup.  
- **Cons:** Data is not real-time; you need a repeatable process (script) for restore + index creation; reporting is down or on stale data during refresh unless you use two alternating copies.

#### Option C: pg_basebackup to Seed, Then Logical Replication (Hybrid)

- **Step 1 – Seed with pg_basebackup**
  - Take **pg_basebackup** from the primary to the empty server.
  - Restore and **start as a streaming standby** (with `primary_conninfo`) so it replays WAL and catches up to “now”.
  - **Promote** the standby (`pg_ctl promote` or equivalent). It is now a standalone, read-write instance with a full copy of the primary as of promote time.
- **Step 2 – Reporting objects**
  - On this promoted instance: create the 3–4 reporting indexes and the materialized view. Do **not** add these on the primary.
- **Step 3 – Ongoing sync via logical replication**
  - On primary: ensure `wal_level = logical` and create a **publication** for the same base tables.
  - On the reporting server: **CREATE SUBSCRIPTION** to that publication with **`copy_data = false`** (tables already exist and are in sync from the physical copy).
  - From then on, the reporting server receives only incremental changes from the primary and stays near real-time.
- **Pros:** Fast initial load (physical copy); then continuous sync without full re-restore.  
- **Cons:** More steps; requires one-time promotion and correct subscription setup with `copy_data = false`.

**Recommendation:**  
- Prefer **Option A** if the initial logical copy time is acceptable.  
- Use **Option B** if you explicitly want only pg_basebackup and can accept periodic refresh (e.g. nightly) and a refresh + index-creation procedure.  
- Use **Option C** if the database is very large and you want both a fast initial seed (pg_basebackup) and ongoing sync (logical replication).

---

### 6. Implementation Plan (High Level)

#### 6.1. Primary – Enable Logical Replication

1. **Configure WAL level and slots** (if not already set)
   - `wal_level = logical`
   - `max_replication_slots` and `max_wal_senders` sized sufficiently.

2. **Create dedicated replication user**

```sql
CREATE ROLE repl_logical
  WITH REPLICATION LOGIN PASSWORD 'STRONG_PASSWORD';
```

3. **Allow empty server connectivity in `pg_hba.conf`**

```text
host    replication    repl_logical    <EMPTY_SERVER_IP>/32    md5
host    <DB_NAME>      repl_logical    <EMPTY_SERVER_IP>/32    md5
```

4. **Create publication for `mis_newgen` base tables**

```sql
CREATE PUBLICATION mis_pub FOR TABLE
  mis.table1,
  mis.table2,
  mis.table3,
  mis.table4;  -- the 3–4 key tables
```

#### 6.2. Empty Server – Set Up Logical Reporting Replica

5. **Initialize as normal PostgreSQL instance**
   - Do **not** configure as standby; ensure `pg_is_in_recovery() = false`.

6. **Create database and schema**

```sql
CREATE DATABASE <DB_NAME>;
\c <DB_NAME>
CREATE SCHEMA mis;
```

7. **Create matching base tables**
   - Same structure and primary keys as on the primary.

8. **Create subscription to primary**

```sql
CREATE SUBSCRIPTION mis_sub
CONNECTION 'host=<PRIMARY_HOST> port=5432 dbname=<DB_NAME> user=repl_logical password=STRONG_PASSWORD'
PUBLICATION mis_pub
WITH (copy_data = true, create_slot = true, enabled = true);
```

9. **Verify replication is healthy**

```sql
SELECT * FROM pg_stat_subscription;
```

---

### 7. Reporting‑Side Indexing and Materialized View

#### 7.1. Add Indexes Only on Reporting Server

10. **Create the required indexes on the 3–4 tables on the reporting replica only**, for example:

```sql
CREATE INDEX CONCURRENTLY idx_table1_colA ON mis.table1 (colA);
CREATE INDEX CONCURRENTLY idx_table2_colB ON mis.table2 (colB);
-- etc.
```

- These indexes are **not** created on the primary, so primary write performance is preserved.

#### 7.2. Materialized View to Replace `mis_newgen`

11. **Create materialized view on reporting replica**

```sql
CREATE MATERIALIZED VIEW mis.mis_newgen_mv AS
SELECT
  ...
FROM mis.table1 t1
JOIN mis.table2 t2 ON ...
JOIN mis.table3 t3 ON ...
JOIN mis.table4 t4 ON ...;
```

12. **Create unique index to allow concurrent refresh**

```sql
CREATE UNIQUE INDEX CONCURRENTLY mis_newgen_mv_pk
  ON mis.mis_newgen_mv (primary_key_column);
```

13. **Use MV in reports instead of the original view**

```sql
SELECT *
FROM mis.mis_newgen_mv
WHERE <appropriate filters>;
```

#### 7.3. Safe Refresh Strategy

14. **Refresh during off‑peak windows** (on reporting server)

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;
```

15. **Automate refresh** via cron or systemd (e.g. nightly or hourly), based on required data freshness.

---

### 8. Benefits of This Solution

- **No impact on primary write performance:**
  - No additional heavy indexes on primary tables.
  - Primary remains optimized for high insert throughput.

- **Safe and fast reporting:**
  - Heavy joins and large scans are offloaded to the logical reporting replica.
  - Reporting server can use higher `work_mem` and more parallelism.

- **Clear separation of concerns:**
  - Physical standby remains dedicated to HA/failover.
  - Logical replica is dedicated to analytics and indexing.

- **Operational safety:**
  - Logical replication can be disabled or dropped without affecting primary business operations.

---

### 9. Summary

- The current secondary cannot host indexes because it is a **read‑only hot standby**.
- To index `mis_newgen`’s base tables without slowing down writes, we will:
  - Keep the primary minimal and write‑optimized.
  - Keep the existing physical standby for HA.
  - Use the empty server as a **logical reporting replica** with additional indexes and a materialized view (`mis_newgen_mv`).
- All heavy `mis_newgen` reporting will run on the logical replica, preventing memory exhaustion and preserving application performance on the primary.


