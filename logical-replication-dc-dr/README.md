# PostgreSQL 14 Logical Replication on RHEL 9 – Full Guide

This document provides step-by-step instructions to set up **logical replication** in PostgreSQL 14 on Red Hat Enterprise Linux (RHEL) 9.

---

## Table of Contents

- **[DC–DR 2+2 nodes – logical replication (elaborate)](DC-DR-2-NODES-EACH.md)** – Full steps for DC (2 nodes: primary + streaming standby) and DR (2 nodes: logical subscriber from DC + streaming standby), with RHEL 9 paths, verification, and failover notes.

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [RHEL and PostgreSQL Setup](#3-rhel-and-postgresql-setup)
4. [Publisher (Source) Configuration](#4-publisher-source-configuration)
5. [Subscriber (Target) Configuration](#5-subscriber-target-configuration)
6. [Creating Publication and Subscription](#6-creating-publication-and-subscription)
7. [Verification and Monitoring](#7-verification-and-monitoring)
8. [Common Operations](#8-common-operations)
9. [Troubleshooting](#9-troubleshooting)
10. [References](#10-references)

---

## 1. Overview

### What is logical replication?

- **Logical replication** copies data changes based on **replication identity** (usually primary key) and **WAL** decoded into logical change records.
- Unlike **streaming (physical) replication**, it allows:
  - Replicating **selected tables** (not the whole cluster).
  - **Different major versions** (e.g. 14 → 15) in some cases.
  - **Multi-source** or **selective** replication to different subscribers.

### Roles

| Role          | Description                          |
|---------------|--------------------------------------|
| **Publisher** | Primary server; publishes changes.    |
| **Subscriber**| Replica server; subscribes and applies changes. |

### Requirements (PostgreSQL 14)

- `wal_level = logical` on the **publisher**.
- Tables must have a **primary key** or **unique** constraint (replication identity).
- **Network connectivity** between publisher and subscriber (default port 5432 or your custom port).
- **RHEL 9**: PostgreSQL 14 installed (e.g. from PostgreSQL Yum repository).

---

## 2. Prerequisites

### 2.1 Software

- **RHEL 9** (or compatible).
- **PostgreSQL 14** on both publisher and subscriber.
- Same PostgreSQL 14 minor version recommended on both sides.

### 2.2 Network and firewall

- Allow **TCP port 5432** (or your `listen_addresses` / port) between publisher and subscriber.
- On RHEL 9 with firewalld:

```bash
# On both servers (adjust zone if needed)
sudo firewall-cmd --permanent --add-service=postgresql
sudo firewall-cmd --reload
```

### 2.3 Users and permissions

- A **superuser** or a user with `CREATE` on database and **replication** privileges to manage publications and subscriptions.
- For the **subscription**, the subscriber connects to the publisher; the publisher user needs:
  - `LOGIN`
  - `REPLICATION` (for logical replication connection), or membership in a role that has it.
  - `SELECT` on published tables (for initial table copy and refresh).

Create a dedicated replication user on the **publisher** (optional but recommended):

```sql
-- Run on PUBLISHER as superuser
CREATE USER logical_repl WITH REPLICATION PASSWORD 'your_secure_password';
-- Grant usage on schema and select on tables you will publish (or grant to schema/table owner)
GRANT USAGE ON SCHEMA public TO logical_repl;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO logical_repl;
```

---

## 3. RHEL and PostgreSQL Setup

### 3.1 Typical paths on RHEL 9

- **Data directory**: `/var/lib/pgsql/14/data`
- **Config**: `postgresql.conf`, `pg_hba.conf` under data directory.
- **Service**: `postgresql-14` (systemd).

### 3.2 Ensure PostgreSQL is running

```bash
sudo systemctl status postgresql-14
sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14   # if not running
```

### 3.3 Locate configuration files

```bash
sudo -u postgres psql -t -c "SHOW config_file;"
sudo -u postgres psql -t -c "SHOW hba_file;"
```

---

## 4. Publisher (Source) Configuration

### 4.1 Set `wal_level = logical`

1. Edit `postgresql.conf` (as root or postgres):

```bash
sudo vi /var/lib/pgsql/14/data/postgresql.conf
```

2. Set or add:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

3. Restart PostgreSQL:

```bash
sudo systemctl restart postgresql-14
```

4. Verify:

```sql
SHOW wal_level;           -- must be 'logical'
SHOW max_replication_slots;
SHOW max_wal_senders;
```

### 4.2 Allow replication connections in `pg_hba.conf`

Add a line so the **subscriber** can connect as the replication user (replace IP with subscriber IP or subnet):

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             logical_repl    <subscriber_ip>/32      scram-sha-256
# Or for a subnet:
host    all             logical_repl    192.168.1.0/24          scram-sha-256
```

Reload configuration (no restart needed):

```bash
sudo systemctl reload postgresql-14
```

### 4.3 Create database and tables (if not already present)

Logical replication works per **database**. Ensure the database and tables exist on the publisher:

```sql
CREATE DATABASE myapp;
\c myapp
CREATE TABLE t1 (id int PRIMARY KEY, data text);
-- Add more tables as needed
```

---

## 5. Subscriber (Target) Configuration

### 5.1 Create the same database and table structure

The **schema** (table definitions) must exist on the subscriber. Data can be copied by the subscription (initial sync) or pre-seeded.

1. Create database:

```sql
CREATE DATABASE myapp;
\c myapp
```

2. Create tables with the **same names and columns** as on the publisher (same primary key/unique constraints):

```sql
CREATE TABLE t1 (id int PRIMARY KEY, data text);
```

You can use `pg_dump -s` (schema only) from the publisher and restore on the subscriber to clone structure:

```bash
# On publisher
pg_dump -h publisher_host -U postgres -s -d myapp > myapp_schema.sql

# Copy myapp_schema.sql to subscriber, then:
psql -h localhost -U postgres -d myapp -f myapp_schema.sql
```

### 5.2 Subscriber PostgreSQL settings

No `wal_level = logical` required on the subscriber for receiving the subscription. Ensure:

- `max_logical_replication_workers` (default 4) is sufficient if you have many subscriptions.
- `max_worker_processes` is high enough (default is usually fine).

Optional (for many subscriptions):

```conf
max_logical_replication_workers = 8
max_worker_processes = 16
```

Restart if you change these.

---

## 6. Creating Publication and Subscription

### 6.1 On publisher: create publication

Connect to the **publisher** database that contains the tables to replicate:

```sql
\c myapp

-- Publish specific tables
CREATE PUBLICATION mypub FOR TABLE t1, t2;

-- Or publish all tables in a schema
CREATE PUBLICATION mypub FOR ALL TABLES IN SCHEMA public;

-- Or publish all tables in the database
CREATE PUBLICATION mypub FOR ALL TABLES;
```

List publications:

```sql
\dRp+
```

### 6.2 On subscriber: create subscription

Connect to the **subscriber** database:

```sql
\c myapp

CREATE SUBSCRIPTION mysub
CONNECTION 'host=publisher_host port=5432 dbname=myapp user=logical_repl password=your_secure_password'
PUBLICATION mypub;
```

- Replace `publisher_host` with the publisher IP or hostname.
- Use the user that has `REPLICATION` and `SELECT` on published tables.

List subscriptions:

```sql
\dRs+
```

### 6.3 Initial data sync

- When you create the subscription, PostgreSQL 14 **copies initial table data** from publisher to subscriber by default (unless you create the subscription with `WITH (copy_data = false)`).
- For large tables, this can take time. Progress can be seen in the subscriber logs and by checking row counts on both sides.

Check replication status:

```sql
-- On subscriber
SELECT * FROM pg_stat_subscription;
SELECT subname, subenabled, subconninfo FROM pg_subscription;
```

---

## 7. Verification and Monitoring

### 7.1 Publisher: check replication slots

```sql
SELECT slot_name, plugin, slot_type, active, restart_lsn
FROM pg_replication_slots;
```

Logical subscriptions create a **replication slot** on the publisher. Ensure `active = t` when the subscriber is connected.

### 7.2 Subscriber: check subscription and lag

```sql
SELECT * FROM pg_stat_subscription;
```

Check application logs on the subscriber for errors:

```bash
sudo journalctl -u postgresql-14 -f
```

### 7.3 Compare row counts (sanity check)

Run on both publisher and subscriber (same database name):

```sql
SELECT relname, n_live_tup
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY relname;
```

---

## 8. Common Operations

### 8.1 Add a table to an existing publication

On **publisher**:

```sql
ALTER PUBLICATION mypub ADD TABLE new_table;
```

Then on **subscriber**, create the table (same structure) and add it to the subscription:

```sql
CREATE TABLE new_table ( ... );  -- same as publisher
ALTER SUBSCRIPTION mysub REFRESH PUBLICATION;
```

### 8.2 Remove a table from publication

On publisher:

```sql
ALTER PUBLICATION mypub DROP TABLE old_table;
```

The table remains on the subscriber; it just stops receiving new changes.

### 8.3 Pause and resume subscription

On subscriber:

```sql
ALTER SUBSCRIPTION mysub DISABLE;
ALTER SUBSCRIPTION mysub ENABLE;
```

### 8.4 Drop subscription (on subscriber)

```sql
DROP SUBSCRIPTION mysub;
```

This also drops the replication slot on the publisher. Drop the publication on the publisher if no longer needed:

```sql
DROP PUBLICATION mypub;
```

---

## 9. Troubleshooting

### 9.1 Subscription not receiving changes

- Check **pg_hba.conf** on publisher allows the subscriber IP and replication user.
- Check **replication slot** on publisher: `pg_replication_slots` should show `active = t`.
- Check subscriber logs: `journalctl -u postgresql-14`.

### 9.2 Replication slot inactive / lagging

- Ensure the subscriber is running and can reach the publisher.
- Check network and firewall (port 5432).
- On subscriber: `pg_stat_subscription` and logs for apply errors.

### 9.3 Conflicts (e.g. duplicate key)

- Logical replication expects **no conflicting writes** on the subscriber for replicated tables.
- Resolve by fixing data on subscriber or skipping conflicting transactions (advanced). Ensure only the subscription applies changes to replicated tables, or use conflict handling (PostgreSQL 15+ has more options).

### 9.4 Permission denied on publisher

- Replication user needs `REPLICATION` and `SELECT` on published tables (and schema).
- Re-check `GRANT` on publisher for the user used in the subscription `CONNECTION` string.

---

## 10. References

- [PostgreSQL 14 Logical Replication](https://www.postgresql.org/docs/14/logical-replication.html)
- [PostgreSQL 14 Create Publication](https://www.postgresql.org/docs/14/sql-createpublication.html)
- [PostgreSQL 14 Create Subscription](https://www.postgresql.org/docs/14/sql-createsubscription.html)
- RHEL 9: [Managing PostgreSQL](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_databases_in_rhel_9/managing-postgresql_managing-databases-in-rhel-9)

---

*Document version: 1.0 | PostgreSQL 14 | RHEL 9*
