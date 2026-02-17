## PostgreSQL 14 Write-Heavy Architecture with Logical Reporting Replica

This document describes a **production-safe architecture** for a write-heavy PostgreSQL 14 system on RHEL 9 where:

- A `mis_newgen`-style view can produce ~500M rows.
- Ad-hoc `SELECT *` can exhaust ~56GB of RAM.
- Missing/reporting indexes **cannot** be added on the primary due to insert performance concerns.
- A secondary empty server is available and will be used as a dedicated reporting node.

The design uses **logical replication** from the primary (OLTP) node to a reporting node, where heavy indexes and materialized views live, leaving the primary optimized for writes.

---

## 1. High-Level Architecture

- **Primary (OLTP)**
  - PostgreSQL 14, RHEL 9.
  - `wal_level = logical` to support logical replication.
  - Tables optimized for **insert/update throughput**, with **only necessary indexes**.
  - Strict protections against large ad-hoc queries (`statement_timeout`, smaller `work_mem` per role).

- **Reporting Server (Logical Subscriber)**
  - PostgreSQL 14, RHEL 9.
  - Subscribes via logical replication to specific tables (those that underpin `mis_newgen`).
  - Has additional **reporting indexes**, **materialized views**, and higher `work_mem`.
  - All heavy reporting / analytical queries run here instead of on primary.

- **Optional HA**
  - A separate **physical streaming replica** can be used for HA/failover of the primary.
  - That HA replica mirrors primary’s schema and index set.
  - The reporting node is **not** used for failover; it is for analytics only.

---

## 2. Logical Replication Setup (Primary → Reporting)

### 2.1. Assumptions and Names

- Database: `prod_db`
- Schema: `mis`
- Base tables for the view: e.g. `mis.fact_txn`, `mis.dim_customer`, `mis.dim_product`.
- Primary host: `primary-db.prod.local`
- Reporting host: `report-db.prod.local`
- Replication user: `repl_logical`

Adjust these names to match your environment.

### 2.2. Primary Configuration

In `postgresql.conf` on the primary:

- **Logical replication parameters**
  - `wal_level = logical`
  - `max_replication_slots = 10`        -- higher if you expect more subscribers
  - `max_wal_senders = 10`
  - `max_logical_replication_workers = 8`
  - `max_worker_processes = 16`
  - `max_sync_workers_per_subscription = 4`

In `pg_hba.conf` on primary, allow the reporting host to connect:

```text
host    replication    repl_logical    report-db.prod.local/32    md5
host    prod_db        repl_logical    report-db.prod.local/32    md5
```

Reload configuration (RHEL 9 / systemd):

```bash
sudo systemctl reload postgresql-14
```

Create the logical replication user on the primary:

```sql
CREATE ROLE repl_logical
  WITH REPLICATION LOGIN PASSWORD 'CHANGE_ME_STRONG_PASSWORD';
```

### 2.3. Create Publication on Primary

Identify all base tables used by `mis_newgen` and add them to a publication.

Example (explicit table list):

```sql
CREATE PUBLICATION mis_pub FOR TABLE
  mis.fact_txn,
  mis.dim_customer,
  mis.dim_product;
```

Or, for an entire schema:

```sql
CREATE PUBLICATION mis_pub FOR TABLES IN SCHEMA mis;
```

### 2.4. Reporting Server: Initial Setup and Subscription

Create the database if needed:

```sql
CREATE DATABASE prod_db;
```

Connect to `prod_db` on the reporting server, and create schemas and tables mirroring the primary (structure must be compatible; names, columns, and primary keys must align).

Example:

```sql
CREATE SCHEMA mis;

CREATE TABLE mis.fact_txn (
  id           bigint PRIMARY KEY,
  customer_id  bigint NOT NULL,
  product_id   bigint NOT NULL,
  txn_ts       timestamptz NOT NULL,
  amount       numeric(18,2) NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE mis.dim_customer (
  id        bigint PRIMARY KEY,
  segment   text NOT NULL,
  -- other columns...
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE mis.dim_product (
  id        bigint PRIMARY KEY,
  category  text NOT NULL,
  -- other columns...
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Create the subscription on the reporting server:

```sql
CREATE SUBSCRIPTION mis_sub
CONNECTION 'host=primary-db.prod.local port=5432 dbname=prod_db user=repl_logical password=CHANGE_ME_STRONG_PASSWORD'
PUBLICATION mis_pub
WITH (
  copy_data = true,
  create_slot = true,
  enabled = true
);
```

Check subscription status:

```sql
SELECT * FROM pg_stat_subscription;
```

Once initial copy completes, the reporting server will follow the primary in near real time.

---

## 3. Logical vs Streaming Replication in This Scenario

### 3.1. Streaming Replication (Physical)

- **Pros**
  - Simple, block-level replication with relatively low overhead.
  - Great for HA / standby failover.
  - Replica is read-only “hot standby” for read scaling.

- **Cons in This Use Case**
  - Standby must remain **physically identical** to primary: same schema, same indexes.
  - You **cannot** create reporting-only indexes or materialized views on the standby that modify primary’s physical layout.
  - While you can add read-only reporting queries, they compete for cache and I/O with primary-like layout.

**Conclusion:**  
Streaming replication is ideal for HA, but **not ideal as your reporting node** when you must avoid extra indexes on the primary.

### 3.2. Logical Replication

- **Pros**
  - Replicates table/row changes, not blocks.
  - Allows **schema divergence** on subscriber:
    - You can add extra reporting indexes.
    - You can create and maintain materialized views, summary tables, or denormalized schemas.
  - Subscriber is read-write for:
    - Non-replicated tables.
    - Reporting-side aggregates and metadata.

- **Cons**
  - More WAL overhead due to logical-change information.
  - Requires management of replication slots to avoid excess WAL retention.
  - Some DDL is not fully transparent; you need a deployment process that handles schema changes cleanly.

**Conclusion:**  
For this scenario—write-heavy primary, heavy reporting, extra indexes only on secondary—**logical replication is the right pattern** for the reporting server. Keep streaming replication (optional) for HA of the primary, not for analytics.

---

## 4. WAL Impact Analysis

### 4.1. WAL on the Primary (Baseline)

Each INSERT/UPDATE/DELETE generates WAL proportional to:

- Heap tuple size.
- Number and type of indexes.
- Full-page writes at checkpoints.

Approximate WAL per insert:

\[
WAL_{\text{insert}} \approx S_{\text{heap}} + \sum_{i=1}^{k} S_{\text{idx}_i} + O_{\text{overhead}}
\]

Where:

- \( S_{\text{heap}} \) is row size in bytes.
- \( S_{\text{idx}_i} \) is per-index entry size.
- \( k \) is number of indexes.
- \( O_{\text{overhead}} \) includes metadata, page headers, etc.

### 4.2. Extra WAL from Logical Replication

With `wal_level = logical`:

- WAL records carry additional information to reconstruct logical changes (keys, old/new tuples as required).
- A logical replication slot is created per subscription.

Implications:

- **WAL retention:** WAL segments cannot be recycled past the replication slot’s `restart_lsn`. If the subscriber is slow or offline, WAL will accumulate on disk.
- **Decoding overhead:** Logical decoding processes read and decode WAL to feed subscribers.

Monitor retained WAL:

```sql
SELECT
  slot_name,
  active,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

### 4.3. Mitigation Strategies

- Ensure reporting server has enough CPU/IO to **keep up**.
- Set `max_wal_size` high enough for peak lag.
- Monitor `pg_replication_slots` and alert when `retained_wal` crosses a threshold.
- If necessary, temporarily **disable heavy reporting queries** during catch-up.

---

## 5. Write Amplification and Indexes

For each table with \( k \) indexes:

- **INSERT**
  - Insert row into heap.
  - Insert one entry per index.
  - Total cost grows roughly linearly with \( k \).

- **UPDATE**
  - If indexed columns change:
    - Index entries deleted + reinserted.
  - If only non-indexed columns change and HOT is possible:
    - Cheaper, but still uses WAL.

- **DELETE**
  - Remove index entries and mark heap tuple dead.

The more (and wider) indexes:

- The **more WAL** is produced.
- The **more CPU and I/O** per write.
- The **more autovacuum work** is required to keep indexes healthy.

**Design Implication:**

- On primary: keep only the **minimal index set required** for OLTP queries and constraints.
- On reporting server: place **heavy analytical indexes** and materialized views where they do not impact OLTP write performance.

---

## 6. Hash Join Memory Model

For a hash join, PostgreSQL builds a hash table on the **build side** and probes it with the other side.

Let:

- \( N_b \): number of rows on build side.
- \( W_b \): average payload bytes per row in the hash table (only columns that executor carries).
- \( O_t \): per-tuple overhead (structs, alignment; typically ~40–80 bytes).
- \( F \): hash-table overhead factor (buckets, load factor; ~1.3–2.0).
- `work_mem`: per-node memory budget.
- `hash_mem_multiplier`: (PG14 default 1.0) multiplier for hash operations.

Approximate in-memory size:

\[
M_{\text{hash}} \approx N_b \times (W_b + O_t) \times F
\]

Hash stays in memory if:

\[
M_{\text{hash}} \le \text{work\_mem} \times \text{hash\_mem\_multiplier}
\]

If this is exceeded, PostgreSQL performs multi-batch disk-backed hash joins (more I/O, slower).

**Rule of Thumb Implementation:**

```text
estimated_hash_mem_bytes
  ≈ rows_build * (avg_row_bytes + 64) * 1.5
```

On the reporting server, you can safely use larger `work_mem` (per reporting role) to keep typical hash joins fully in memory, while keeping `work_mem` small on the primary.

---

## 7. Primary Server Configuration (Write-Heavy OLTP)

Assume ~56GB RAM (adjust to your real value).

### 7.1. Memory

- `shared_buffers = 14GB`              -- ≈25% RAM
- `effective_cache_size = 42GB`        -- ≈75% RAM
- `work_mem = 16MB`                    -- conservative global default
- `maintenance_work_mem = 2GB`         -- for VACUUM/REINDEX, but concurrency-aware

For OLTP application role:

```sql
ALTER ROLE app_user SET work_mem = '8MB';
ALTER ROLE app_user SET statement_timeout = '30s';
```

### 7.2. WAL and Checkpoints

- `wal_level = logical`
- `synchronous_commit = on` (or `local` if you can slightly relax durability)
- `wal_compression = on`
- `max_wal_size = 64GB`
- `min_wal_size = 8GB`
- `checkpoint_timeout = '15min'`
- `checkpoint_completion_target = 0.9`

### 7.3. Autovacuum

- `autovacuum = on`
- `autovacuum_max_workers = 5`
- `autovacuum_naptime = '10s'`
- `autovacuum_vacuum_scale_factor = 0.05`
- `autovacuum_analyze_scale_factor = 0.02`

Per-table tuning for write-heavy tables:

```sql
ALTER TABLE mis.fact_txn SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01
);
```

### 7.4. Connections and Parallelism

- `max_connections = 300` (prefer smaller if using pgbouncer)
- `max_worker_processes = 16`
- `max_parallel_workers = 8`
- `max_parallel_workers_per_gather = 2`

Protect primary from runaway analytics:

- Use smaller `work_mem` and strict `statement_timeout` for OLTP roles.
- Optionally set `idle_in_transaction_session_timeout` to avoid stuck sessions.

---

## 8. Reporting Server Configuration (Analytics)

### 8.1. Memory

- `shared_buffers = 16GB`
- `effective_cache_size = 40GB`
- `work_mem = 128MB` (global; adjust based on concurrency)

For reporting roles:

```sql
ALTER ROLE report_user SET work_mem = '512MB';
ALTER ROLE report_user SET statement_timeout = '2h';
```

### 8.2. WAL and Checkpoints

- `wal_level = replica` (logical not needed unless you further replicate from reporting)
- `synchronous_commit = off` (acceptable since this is a derived/reporting copy)
- `wal_compression = on`
- `max_wal_size = 32GB`

### 8.3. Parallelism

- `max_parallel_workers = 16`
- `max_parallel_workers_per_gather = 4`
- `parallel_leader_participation = on`

### 8.4. Subscription Tuning

On reporting server:

```sql
ALTER SUBSCRIPTION mis_sub SET (
  copy_data = false
);

ALTER SUBSCRIPTION mis_sub ENABLE;
```

(If your exact PostgreSQL 14 minor version supports `streaming` for logical initial copy, you can add `streaming = on`; otherwise omit.)

---

## 9. Materialized View Strategy for `mis_newgen`

### 9.1. Replace Heavy View with MV on Reporting

On the **reporting server**, create a materialized view that encapsulates the `mis_newgen` logic:

```sql
CREATE MATERIALIZED VIEW mis.mis_newgen_mv AS
SELECT
  f.id,
  f.txn_ts,
  f.amount,
  c.segment,
  p.category
  -- other required columns/derived fields...
FROM mis.fact_txn      f
JOIN mis.dim_customer  c ON c.id = f.customer_id
JOIN mis.dim_product   p ON p.id = f.product_id;
```

Add a unique index required for concurrent refresh:

```sql
CREATE UNIQUE INDEX CONCURRENTLY mis_newgen_mv_pk
  ON mis.mis_newgen_mv (id);
```

Users then query:

```sql
SELECT *
FROM mis.mis_newgen_mv
WHERE txn_ts >= now() - interval '1 day';
```

### 9.2. Refresh Strategy with `REFRESH MATERIALIZED VIEW CONCURRENTLY`

PostgreSQL 14 does not support fully automatic incremental refresh, but you can:

- Do **full concurrent refresh** during off-peak times:

  ```sql
  REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;
  ```

- Partition underlying large tables (e.g., `fact_txn` by day/month) to shrink the effective working set.
- Optionally maintain smaller, **time-windowed MVs** (e.g., last 7 days) for most queries.

---

## 10. Indexing Strategy on Reporting Server

On the reporting server only:

- **Indexes on MV** (example patterns):

```sql
CREATE INDEX CONCURRENTLY mis_newgen_mv_txn_ts_idx
  ON mis.mis_newgen_mv (txn_ts);

CREATE INDEX CONCURRENTLY mis_newgen_mv_segment_ts_idx
  ON mis.mis_newgen_mv (segment, txn_ts DESC);

CREATE INDEX CONCURRENTLY mis_newgen_mv_category_ts_idx
  ON mis.mis_newgen_mv (category, txn_ts DESC);
```

- **Indexes on base tables** (if queried directly for ad-hoc reporting):
  - Partition `fact_txn` by date.
  - Create local indexes on `(txn_ts)` and join keys `(customer_id)`, `(product_id)` as needed.

- Avoid `SELECT *`:
  - Create **narrow views** for specific report use-cases.
  - This reduces row width and memory usage for joins and sorts.

---

## 11. Risk Analysis and Rollback

### 11.1. Key Risks

- **Replication Lag (Reporting Behind Primary)**
  - Heavy analytical queries on reporting can delay apply of replication changes.
  - Users might see stale data in reports.

- **Excess WAL Retention on Primary**
  - If reporting server is down/slow, logical slot retains WAL segments on the primary.
  - May consume substantial disk space.

- **Schema Drift**
  - DDL changes on primary must be coordinated with subscriber.
  - Some changes may require manual updates on reporting server.

- **Resource Contention on Reporting**
  - Underestimating concurrency and `work_mem` may still cause memory pressure and swapping.

### 11.2. Monitoring Lag and WAL

On primary (if you also have physical replicas):

```sql
SELECT
  pid,
  application_name,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replication_lag
FROM pg_stat_replication;
```

On reporting (logical subscription):

```sql
SELECT
  subname,
  now() - last_msg_receipt_time AS apply_delay
FROM pg_stat_subscription;
```

On primary, monitor logical slots:

```sql
SELECT
  slot_name,
  active,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

### 11.3. Rollback Plan

If logical replication or reporting node causes unacceptable impact:

1. **Disable subscription on reporting:**

   ```sql
   ALTER SUBSCRIPTION mis_sub DISABLE;
   ```

2. **Drop subscription (optional):**

   ```sql
   DROP SUBSCRIPTION mis_sub;
   ```

   This also drops the replication slot on the primary (if created by subscription and `DROP` is successful).

3. **Clean up reporting-specific objects (optional):**
   - Drop materialized views and indexes if you want to fully revert.

4. **Primary impact relief:**
   - Confirm that WAL retention shrinks (using `pg_replication_slots`).
   - Adjust parameters (`max_wal_size`, disk alerts) if needed.

The primary continues operating; logical replication is additive and can be safely removed without affecting core OLTP operations.

---

## 12. Summary

- Use **logical replication** to a **dedicated reporting server** where:
  - Heavy indexes and materialized views live.
  - Large `SELECT` queries no longer threaten primary memory.
- Keep the primary optimized for **writes**, with minimal indexing and protective timeouts.
- Use **materialized views plus concurrent refresh** on the reporting node to expose a `mis_newgen`-like structure safely.
- Monitor replication lag and WAL retention, and maintain a clear rollback path (disable/drop subscription) for operational safety.

