---

# Logical Replication with Multiple Databases (PostgreSQL)

Logical Replication (LR) works **per database**.
If you have **2 or more databases**, you must configure replication **separately for each database**.

---

## What Changes When You Have 2 Databases

| Location       | Action Required                                                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **DC-Primary** | Create **one publication per database** (e.g., `dc_to_dr_pub_app1`, `dc_to_dr_pub_app2`). Grant `logical_repl` user schema USAGE and SELECT privileges in each database. |
| **DR-Primary** | Create **one subscription per database**, inside the respective database, pointing to the matching publication.                                                          |

---

## Example: Two Databases (`app1`, `app2`)

---

### On DC-Primary

#### Database: app1

```sql
\c app1

CREATE USER logical_repl WITH REPLICATION PASSWORD 'xxx';

GRANT USAGE ON SCHEMA public TO logical_repl;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;

CREATE PUBLICATION dc_to_dr_pub_app1 FOR ALL TABLES IN SCHEMA public;
```

#### Database: app2

```sql
\c app2

-- User already exists (cluster-wide), only grant privileges
GRANT USAGE ON SCHEMA public TO logical_repl;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;

CREATE PUBLICATION dc_to_dr_pub_app2 FOR ALL TABLES IN SCHEMA public;
```

---

### On DR-Primary

1. Create databases `app1` and `app2`
2. Restore **schema only** from DC:

```bash
pg_dump -s -d app1 > app1_schema.sql
pg_dump -s -d app2 > app2_schema.sql
```

3. Create subscriptions:

#### Database: app1

```sql
\c app1

CREATE SUBSCRIPTION dc_to_dr_sub_app1
CONNECTION 'host=dc-primary.example.com port=5432 dbname=app1 user=logical_repl password=xxx'
PUBLICATION dc_to_dr_pub_app1
WITH (copy_data = true);
```

#### Database: app2

```sql
\c app2

CREATE SUBSCRIPTION dc_to_dr_sub_app2
CONNECTION 'host=dc-primary.example.com port=5432 dbname=app2 user=logical_repl password=xxx'
PUBLICATION dc_to_dr_pub_app2
WITH (copy_data = true);
```

---

### pg_hba.conf on DC-Primary

Only **one entry is required** for the replication user (applies to all databases):

```conf
host    all    logical_repl    <DR_IP>/32    md5
```

---

## Key Points (Short Answer)

* ✅ Logical replication **supports multiple databases**
* 📌 **One publication per database** on DC
* 📌 **One subscription per database** on DR
* 👤 Same `logical_repl` user can be reused (grant privileges per DB)
* ⚠ Physical streaming replication replicates **entire cluster**, logical replication replicates **selected tables per database**

---
