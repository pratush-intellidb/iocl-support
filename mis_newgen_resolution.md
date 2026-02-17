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
- Use the empty server as a **logical reporting replica**, where we:
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

### 5. Implementation Plan (High Level)

#### 5.1. Primary – Enable Logical Replication

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

#### 5.2. Empty Server – Set Up Logical Reporting Replica

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

### 6. Reporting‑Side Indexing and Materialized View

#### 6.1. Add Indexes Only on Reporting Server

10. **Create the required indexes on the 3–4 tables on the reporting replica only**, for example:

```sql
CREATE INDEX CONCURRENTLY idx_table1_colA ON mis.table1 (colA);
CREATE INDEX CONCURRENTLY idx_table2_colB ON mis.table2 (colB);
-- etc.
```

- These indexes are **not** created on the primary, so primary write performance is preserved.

#### 6.2. Materialized View to Replace `mis_newgen`

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

#### 6.3. Safe Refresh Strategy

14. **Refresh during off‑peak windows** (on reporting server)

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;
```

15. **Automate refresh** via cron or systemd (e.g. nightly or hourly), based on required data freshness.

---

### 7. Benefits of This Solution

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

### 8. Summary

- The current secondary cannot host indexes because it is a **read‑only hot standby**.
- To index `mis_newgen`’s base tables without slowing down writes, we will:
  - Keep the primary minimal and write‑optimized.
  - Keep the existing physical standby for HA.
  - Use the empty server as a **logical reporting replica** with additional indexes and a materialized view (`mis_newgen_mv`).
- All heavy `mis_newgen` reporting will run on the logical replica, preventing memory exhaustion and preserving application performance on the primary.


