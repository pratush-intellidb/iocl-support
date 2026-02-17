## PostgreSQL 14 Reporting Ops, Automation, and Hardening

This document complements `postgres14_reporting_architecture.md` with:

- Materialized view refresh strategy and cron/job examples.
- Index maintenance notes for the reporting server.
- Operational monitoring queries.
- Production hardening checklist.

---

## 1. Materialized View Refresh Strategy and Automation

We assume a materialized view `mis.mis_newgen_mv` on the reporting server as described in the architecture document.

### 1.1. Refresh Command

On the **reporting server**:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;
```

Prerequisites:

- MV has a **unique index** on all rows (e.g., `ON (id)`).
- Refresh is run when system load is acceptable, often off-peak.

### 1.2. Example Refresh Schedules

Common options:

- **Nightly full refresh** (simplest, robust).
- **Hourly refresh** for near-real-time reporting (if MV size + infra allow).

#### 1.2.1. Cron-Based Refresh Using `psql` (Linux/RHEL 9 Pattern)

On the reporting server host, create a script, e.g. `/usr/local/bin/refresh_mis_newgen_mv.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PGHOST="report-db.prod.local"
PGPORT="5432"
PGDATABASE="prod_db"
PGUSER="report_admin"

psql "host=${PGHOST} port=${PGPORT} dbname=${PGDATABASE} user=${PGUSER}" <<'SQL'
REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;
SQL
```

Make it executable:

```bash
chmod +x /usr/local/bin/refresh_mis_newgen_mv.sh
```

Add a cron entry (e.g., nightly at 01:30):

```text
30 1 * * * /usr/local/bin/refresh_mis_newgen_mv.sh >> /var/log/pg_mis_newgen_refresh.log 2>&1
```

#### 1.2.2. `systemd` Timer (Alternative)

Create a systemd service (e.g., `/etc/systemd/system/pg-refresh-mis_newgen.service`):

```ini
[Unit]
Description=Refresh mis.mis_newgen_mv materialized view

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/bin/refresh_mis_newgen_mv.sh
```

Create a timer (e.g., `/etc/systemd/system/pg-refresh-mis_newgen.timer`):

```ini
[Unit]
Description=Nightly refresh of mis.mis_newgen_mv

[Timer]
OnCalendar=*-*-* 01:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now pg-refresh-mis_newgen.timer
```

---

## 2. Indexing and Maintenance Strategy on Reporting

### 2.1. Index Creation Guidelines

On the reporting server:

- Add indexes **based on real query patterns**, especially:
  - Time-based filters (`txn_ts`, date ranges).
  - Segmentation (`segment`, `category`, or other grouping keys).
  - Common join keys.

Example on the MV:

```sql
CREATE INDEX CONCURRENTLY mis_newgen_mv_txn_ts_idx
  ON mis.mis_newgen_mv (txn_ts);

CREATE INDEX CONCURRENTLY mis_newgen_mv_segment_ts_idx
  ON mis.mis_newgen_mv (segment, txn_ts DESC);

CREATE INDEX CONCURRENTLY mis_newgen_mv_category_ts_idx
  ON mis.mis_newgen_mv (category, txn_ts DESC);
```

### 2.2. Reindex and Bloat Management

For large indexes that may bloat over time:

```sql
REINDEX INDEX CONCURRENTLY mis_newgen_mv_txn_ts_idx;
```

Monitor bloat using community queries (pgstattuple, or standard views as available). On PG14, the built-in `pg_stat_user_indexes` is useful for hit ratios and scans:

```sql
SELECT
  schemaname,
  relname,
  indexrelname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 50;
```

---

## 3. Monitoring Queries

### 3.1. Replication Lag (Logical)

On the reporting server:

```sql
SELECT
  subname,
  pid,
  status,
  (now() - last_msg_receipt_time) AS apply_delay,
  (now() - last_msg_send_time) AS send_delay
FROM pg_stat_subscription;
```

On the primary, for logical slots:

```sql
SELECT
  slot_name,
  active,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

If you also use physical streaming replicas:

```sql
SELECT
  pid,
  application_name,
  client_addr,
  state,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replication_lag
FROM pg_stat_replication;
```

### 3.2. Autovacuum and Table Health

Basics:

```sql
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_all_tables
WHERE schemaname = 'mis'
ORDER BY n_dead_tup DESC
LIMIT 50;
```

### 3.3. Long-Running Queries

On either server:

```sql
SELECT
  pid,
  now() - query_start AS duration,
  usename,
  datname,
  state,
  query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC
LIMIT 20;
```

### 3.4. Memory-Heavy Queries (Hash/Sort)

Look for large hash or sort operations:

```sql
SELECT
  pid,
  usename,
  now() - query_start AS duration,
  query
FROM pg_stat_activity
WHERE state = 'active'
  AND query LIKE '%Hash Join%'
ORDER BY duration DESC
LIMIT 20;
```

For deeper analysis, enable `log_min_duration_statement` and review execution plans with `EXPLAIN (ANALYZE, BUFFERS)`.

---

## 4. Automation/Job Examples

### 4.1. MV Refresh (Recap)

Covered in Section 1.2: run nightly/hourly `REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;` via cron or systemd timer.

### 4.2. Regular Statistics Maintenance

Optionally run targeted ANALYZE for critical tables/materialized views:

```bash
#!/usr/bin/env bash
set -euo pipefail

PGHOST="report-db.prod.local"
PGPORT="5432"
PGDATABASE="prod_db"
PGUSER="report_admin"

psql "host=${PGHOST} port=${PGPORT} dbname=${PGDATABASE} user=${PGUSER}" <<'SQL'
ANALYZE VERBOSE mis.fact_txn;
ANALYZE VERBOSE mis.mis_newgen_mv;
SQL
```

Cron (e.g., every night at 02:30):

```text
30 2 * * * /usr/local/bin/analyze_reporting.sh >> /var/log/pg_analyze_reporting.log 2>&1
```

### 4.3. Alerting Thresholds (Conceptual)

Use your monitoring system (Prometheus, Zabbix, etc.) to:

- Alert if:
  - `apply_delay` (logical subscription) > e.g. 5 minutes.
  - `retained_wal` on primary > e.g. 20GB.
  - `disk_usage` on WAL volume > 80%.
  - `long-running queries` on reporting > e.g. 2 hours (or your SLA).

---

## 5. Production Hardening Checklist

### 5.1. Primary (OLTP)

- **Replication & WAL**
  - [ ] `wal_level = logical` configured.
  - [ ] `max_replication_slots` and `max_wal_senders` adequately sized.
  - [ ] WAL volume and retention monitored (`pg_replication_slots`).

- **Performance & Safety**
  - [ ] `shared_buffers`, `effective_cache_size`, `work_mem` sized reasonably.
  - [ ] `statement_timeout` enforced on OLTP roles.
  - [ ] Connection pooling in place (e.g., pgbouncer).
  - [ ] Autovacuum tuned per heavy tables.

- **Security**
  - [ ] Strong password or certificate auth for `repl_logical`.
  - [ ] `pg_hba.conf` restricted to required hosts.
  - [ ] `log_connections`, `log_disconnections`, `log_line_prefix` set for auditability.

- **Backups & HA**
  - [ ] Regular base backups + WAL archiving tested.
  - [ ] (Optional) Physical replica for HA/failover configured and tested.

### 5.2. Reporting Server

- **Logical Replication**
  - [ ] Subscription `mis_sub` created and enabled.
  - [ ] Initial copy completed successfully.
  - [ ] Regular monitoring of `pg_stat_subscription` for `apply_delay`.

- **Schema & Indexes**
  - [ ] Materialized views for heavy reporting (`mis_newgen_mv`) created.
  - [ ] Unique index on MV to support concurrent refresh.
  - [ ] Additional reporting indexes created based on actual query patterns.

- **Resource Management**
  - [ ] Higher `work_mem` for reporting roles, but with concurrency considered.
  - [ ] Query timeouts (`statement_timeout`) set according to SLAs.
  - [ ] Disk capacity planning done for data + indexes + MV.

- **Automation**
  - [ ] Cron or systemd timers for:
    - MV refresh.
    - Periodic ANALYZE (if needed beyond autovacuum).
  - [ ] Log rotation for refresh and maintenance scripts.

- **Security**
  - [ ] Access restricted to reporting users and services.
  - [ ] Separate roles for admin vs. reader accounts.

### 5.3. Rollback and DR

- [ ] Documented procedure to:
  - Disable subscription.
  - Drop subscription and logical slot if necessary.
  - Recreate from scratch if data gets corrupted on reporting server.
- [ ] Tested restore of primary from backup.
- [ ] Clear expectations with stakeholders on:
  - Data freshness on reporting node.
  - Acceptable data loss for reporting in DR scenarios.

---

## 6. Quick Ops Runbook (TL;DR)

- **Check replication health**

  ```sql
  -- On reporting
  SELECT subname, now() - last_msg_receipt_time AS apply_delay
  FROM pg_stat_subscription;
  ```

- **Refresh main reporting MV**

  ```sql
  REFRESH MATERIALIZED VIEW CONCURRENTLY mis.mis_newgen_mv;
  ```

- **Investigate long queries**

  ```sql
  SELECT pid, now() - query_start AS duration, usename, query
  FROM pg_stat_activity
  WHERE state <> 'idle'
  ORDER BY duration DESC
  LIMIT 20;
  ```

- **Rollback logical reporting replica (if required)**
  - `ALTER SUBSCRIPTION mis_sub DISABLE;`
  - Optionally `DROP SUBSCRIPTION mis_sub;`
  - Confirm WAL retention normalizes on primary.

