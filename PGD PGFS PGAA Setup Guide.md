# PGD + PGFS + PGAA Setup Guide

## Cluster Architecture

| Cluster Type | Components |
|---|---|
| **Lakehouse Cluster (Read-Only)** | EPAS + pgfs + pgaa |
| **PGD Cluster (Read-Write)** | EPAS + pgfs + pgaa + pgd (bdr) |

## Node Configuration

### Lab Environment
| IP Address | Hostname | Role |
|---|---|---|
| 192.168.56.43 | warehousepg-m | pgd1 |
| 192.168.56.44 | warehousepg-s1 | pgd2 |
| 192.168.56.45 | warehousepg-s2 | pgd3 |

### Production Environment
| IP Address | Hostname | Role |
|---|---|---|
| 10.0.8.103 | whpg-s | pgd1 |
| 10.0.8.44 | whpg-d1 | pgd2 |
| 10.0.7.130 | whpg-d2 | pgd3 |

---

## Time Synchronization (All Nodes)

```bash
sudo timedatectl status
sudo timedatectl set-ntp true

sudo systemctl status chronyd.service
sudo systemctl restart chronyd.service

sudo systemctl enable --now chronyd
sudo timedatectl set-timezone Asia/Seoul
```

---

## Step 1 — Install on All Nodes

> **User:** `vagrant`

### Configure EDB Repository

```bash
export EDB_SUBSCRIPTION_TOKEN=a42f0873c732be6edafe2e22849eb0ff
export EDB_SUBSCRIPTION_PLAN=enterprise
export EDB_REPO_TYPE=rpm

# Enterprise repository
curl -1sSLf "https://downloads.enterprisedb.com/$EDB_SUBSCRIPTION_TOKEN/enterprise/setup.$EDB_REPO_TYPE.sh" | sudo -E bash

# PGD Expand repository
curl -1sSLf "https://downloads.enterprisedb.com/$EDB_SUBSCRIPTION_TOKEN/postgres_distributed/setup.$EDB_REPO_TYPE.sh" | sudo -E bash
```

### Install EDB Packages

```bash
export PG_VERSION=17

# Choose one of the following package sets:
# export EDB_PACKAGES="edb-as$PG_VERSION-server edb-pgd6-<edition>-epas$PG_VERSION"
export EDB_PACKAGES="edb-as$PG_VERSION-server edb-pgd6-essential-epas$PG_VERSION"
# export EDB_PACKAGES="edb-as$PG_VERSION-server edb-pgd6-essential-epas$PG_VERSION edb-as$PG_VERSION-pgfs edb-as$PG_VERSION-pgaa"

sudo dnf install -y $EDB_PACKAGES
```

### Uninstall (if needed)

```bash
sudo dnf remove -y $EDB_PACKAGES
sudo rm -rf /var/lib/edb/as17/data
```

### Configure enterprisedb User

```bash
echo "enterprisedb" | sudo passwd --stdin "enterprisedb"

sudo -iu enterprisedb

tee -a ~/.bash_profile << EOF
export PG_VERSION=17
export PATH=/usr/edb/as${PG_VERSION}/bin:${PATH}
export PGDATA=/var/lib/edb/as${PG_VERSION}/data/
export PGPASSWORD=secret
export PGUSER=postgres
export PGDATABASE=pgddb
export PGPORT=5432
EOF
```

---

## Step 2 — Configure PGD

### 2-1. Node 1 (pgd1)

```bash
pgd node pgd1 setup \
  --dsn "host=pgd1 user=postgres port=5432 dbname=pgddb" \
  --group-name pgd_gr
```

### 2-2. Node 2 (pgd2)

```bash
pgd node pgd2 setup \
  --dsn "host=pgd2 user=postgres port=5432 dbname=pgddb" \
  --cluster-dsn "host=pgd1 user=postgres port=5432 dbname=pgddb"
```

### 2-3. Node 3 (pgd3)

```bash
pgd node pgd3 setup \
  --dsn "host=pgd3 user=postgres port=5432 dbname=pgddb" \
  --cluster-dsn "host=pgd1 user=postgres port=5432 dbname=pgddb"
```

---

## Step 3 — Verification

### On Node 1

```bash
sudo -iu enterprisedb
export PGUSER=postgres
export PGPORT=5432

/usr/edb/as17/bin/psql -d pgddb
```

```sql
-- Create test table and insert data
CREATE TABLE quicktest ( id SERIAL PRIMARY KEY, value INT );
INSERT INTO quicktest (value) SELECT random()*10000 FROM generate_series(1,10000);

-- Check replication rates
SELECT * FROM bdr.node_replication_rates;

-- Verify row count and sum
SELECT COUNT(*), SUM(value) FROM quicktest;

-- Wait for replication confirmation
SELECT bdr.wait_slot_confirm_lsn(NULL, NULL);
```

### On Nodes 2 and 3

```bash
sudo -iu enterprisedb
export PGUSER=postgres
export PGPORT=5432
/usr/edb/as17/bin/psql -d pgddb
```

```sql
SELECT * FROM bdr.node_replication_rates;
SELECT COUNT(*), SUM(value) FROM quicktest;
SELECT node_name, peer_state_name FROM bdr.node_summary;
```

### Service Management

```bash
sudo systemctl enable edb-as-17.service
sudo systemctl status edb-as-17.service
```

---

## Step 4 — Connection Manager

### Direct Node Connections

```bash
psql "host=pgd2 port=5432 user=postgres dbname=pgddb"

/usr/edb/as17/bin/psql -h pgd1
/usr/edb/as17/bin/psql -h pgd2
/usr/edb/as17/bin/psql -h pgd3
```

```sql
INSERT INTO quicktest (value) SELECT random()*10000 FROM generate_series(1,10000);
SELECT COUNT(*), SUM(value) FROM quicktest;
```

### Connect via Connection Manager — Read/Write (port 6432)

```bash
# Direct connection to a data node
/usr/edb/as17/bin/psql -h pgd1 -p 5432 -U enterprisedb -d pgddb

# Connection Manager (RW)
psql -h pgd3 -p 6432 -U postgres -d pgddb
```

```sql
SELECT bdr.bdr_version();
SELECT node_name FROM bdr.local_node_summary;
SELECT * FROM bdr.local_node_summary;
```

### Connect via Connection Manager — Read-Only (port 6433)

```bash
psql -h pgd1 -p 6433 -U postgres -d pgddb
```

```sql
-- Check the currently connected node
SELECT node_name FROM bdr.local_node_summary;
```

### Routing & Connection Manager Views

```sql
SELECT * FROM bdr.node_group_routing_config_summary;
SELECT * FROM bdr.node_routing_config_summary;
SELECT * FROM bdr.node_group_summary;
SELECT * FROM bdr.node_group_routing_summary;

-- Active session activity (extends pg_stat_activity)
SELECT * FROM bdr.stat_activity;
SELECT pid, leader_pid, usename, xact_start, query_start,
       wait_event_type, wait_event, state
FROM bdr.stat_activity;

-- Overall Connection Manager status and performance metrics
SELECT * FROM bdr.stat_connection_manager;

-- Detailed statistics for each active connection in Connection Manager
SELECT * FROM bdr.stat_connection_manager_connections;

-- Per-node metrics for Connection Manager activity across the cluster
SELECT * FROM bdr.stat_connection_manager_node_stats;

-- Active HBA rules applied by Connection Manager on the local node
SELECT * FROM bdr.stat_connection_manager_hba_file_rules;
```

### Node and Replication Slot Queries

```sql
-- Node information
SELECT node_id, node_name, node_group_id, seq_id, source_node_id, dbname
FROM bdr.node ORDER BY seq_id;

-- Check inactive replication slots
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes,
       restart_lsn
FROM pg_replication_slots
WHERE active = 'f'; -- 'f' = false (inactive)

-- Sequence allocation status
SELECT seqid, seq_chunk_size, seq_allocated_up_to, seq_nallocs
FROM bdr.sequence_alloc;

-- Replication status overview
SELECT * FROM bdr.node_replication_rates;

-- BDR parameter settings
SELECT name, setting FROM pg_settings WHERE name LIKE 'bdr.%';

SHOW bdr.global_lock_timeout;
SHOW bdr.default_conflict_detection;
SHOW bdr.default_sequence_kind;
SHOW bdr.default_replica_identity;
SHOW bdr.ddl_replication;
SHOW bdr.role_replication;
SHOW bdr.ddl_locking;
```

---

## Step 5 — PGD CLI

```bash
sudo usermod -aG wheel enterprisedb

sudo mkdir -p /etc/edb/pgd-cli

cat <<EOF | sudo tee /etc/edb/pgd-cli/pgd-cli-config.yml
cluster:
  name: pgd
  endpoints:
    - host=pgd1 dbname=pgddb port=5432
    - host=pgd2 dbname=pgddb port=5432
    - host=pgd3 dbname=pgddb port=5432
EOF
```

### Common CLI Commands

```bash
pgd nodes list
pgd node <pgd1> show
pgd cluster show --health
pgd cluster show
pgd cluster verify
pgd nodes list --versions
pgd groups list
pgd group <pgd_gr> show --summary
pgd raft show
```

### Group Options

```bash
# Set / get a group option
pgd group pgd_gr set-option location London
pgd group pgd_gr get-option location

# Set a preferred leader
pgd group pgd_gr set-leader pgd2
pgd group pgd_gr show --summary
```

### Database & Service Management

```sql
ALTER USER postgres WITH PASSWORD 'secret';
```

```bash
/usr/edb/as17/bin/pg_ctl -D /var/lib/edb/as17/data stop
/usr/edb/as17/bin/pg_ctl -D /var/lib/edb/as17/data start

/usr/edb/as17/bin/pg_ctl -D /var/lib/edb/as/data stop
/usr/edb/as17/bin/pg_ctl -D /var/lib/edb/as/data start
```

---

## PGAA + PGFS Configuration

### Installation

```bash
sudo dnf install edb-as17-pgfs edb-as17-pgaa
```

### Extension Setup (Run on One Node Only)

```bash
sudo su - enterprisedb
/usr/edb/as17/bin/psql -h pgd1
```

```sql
-- 1. Create pgfs first
CREATE EXTENSION IF NOT EXISTS pgfs;

-- 2. Create pgaa
CREATE EXTENSION IF NOT EXISTS pgaa;

SELECT * FROM pg_extension;
```

### ⚠️ Important: Set BDR Analytics Storage Location (Starts Seafowl Engine)

```sql
-- Identify your group name
SELECT node_group_name FROM bdr.node_group WHERE node_group_name != 'world';

-- Enable raft and routing (replace 'pgd_gr' with your group name)
SELECT bdr.alter_node_group_option('pgd_gr', 'enable_raft', 'on');
SELECT bdr.alter_node_group_option('pgd_gr', 'enable_routing', 'on');

-- Set the analytics storage location
SELECT bdr.alter_node_group_option(
  'pgd_gr',
  'analytics_storage_location',
  'minio_storage'
);

-- Enable proxy routing
SELECT bdr.alter_node_group_option(
  (SELECT node_group_name FROM bdr.node_group
   WHERE node_group_name != 'world' LIMIT 1),
  'enable_proxy_routing', 'true'
);
```

### Verify Seafowl is Running

```sql
SELECT application_name FROM pg_stat_activity WHERE application_name LIKE '%seafowl%';
```

Expected output:
```
          application_name
-------------------------------------
 pgd seafowl_metastore_agent 16384:0
 pgd seafowl_manager 16384:0
(2 rows)
```

> **Note:** `pgd seafowl_manager` must be present for analytics to function.

---

## Storage Location Setup

### Create MinIO Storage Location

```sql
SELECT pgfs.create_storage_location(
    'minio_storage',
    's3://pgaa',
    '{"endpoint": "http://192.168.56.51:9000",
       "allow_http": "true"}',
    '{"access_key_id": "minioadmin",
      "secret_access_key": "password"}'
);
```

### Manage Storage Locations

```sql
-- List all storage locations
SELECT * FROM pgfs.list_storage_locations();

-- Get details of a specific location
SELECT * FROM pgfs.get_storage_location('minio_storage');

-- Delete a storage location
SELECT pgfs.delete_storage_location('minio_storage');

-- Get / set the default storage location
SELECT * FROM pgfs.get_default_storage_location();
SELECT pgfs.set_default_storage_location('minio_storage');

-- Update a storage location
SELECT pgfs.update_storage_location(
  'minio_storage', 's3://my_bucket',
  '123e4567-e89b-12d3-a456-426614174000', '{}', '{}'
);
```

---

## Creating PGAA Tables

### Delta Table (S3/MinIO)

```sql
SET search_path = public, pgaa, pgfs, bdr;

CREATE TABLE IF NOT EXISTS public.pgaa_customer ()
USING PGAA
WITH (
  pgaa.storage_location = 'minio_storage',
  pgaa.path = 'tpch_sf_1/customer',
  pgaa.format = 'delta'
);

SELECT * FROM public.pgaa_customer LIMIT 5;

-- Verify PGAA table mapping
SELECT * FROM pgaa.table_mapping;
```

### Iceberg Table (S3/MinIO)

```sql
CREATE TABLE IF NOT EXISTS public.iceberg_table ()
USING PGAA
WITH (
  pgaa.storage_location = 'minio_storage',
  pgaa.path = 'iceberg-data/default.db/iceberg_table',
  pgaa.format = 'iceberg'
);

SELECT count(*) FROM public.iceberg_table;
```

---

## PGD + PGAA: Replicate to Analytics / Tiered Tables

PGD ships with an embedded analytics engine powered by PGAA, enabling the following capabilities:

### Replicate Transactional Tables to Lakehouse Tables

```sql
CREATE TABLE x(a INT PRIMARY KEY) WITH (pgd.replicate_to_analytics = true);
```

### Query Tables with the Analytics Engine

```sql
SET LOCAL bdr.prefer_analytics_engine = true;
SELECT * FROM x;
```

### Create Tiered Tables (Offload Cold Data to Analytics)

```sql
CREATE TABLE test_part_timestamp (
  a TIMESTAMP PRIMARY KEY,
  b INT
) PARTITION BY RANGE (a);

SELECT bdr.autopartition(
  relation              := 'test_part_timestamp',
  partition_increment   := '5s',
  partition_initial_lowerbound := CURRENT_TIMESTAMP::text,
  managed_locally       := true,
  analytics_offload_period := '10s'
);
```

---

## Troubleshooting

### Error: Arrow IPC / Connection Refused

```
ERROR: arrow error: Ipc error: Status { code: Unavailable,
  message: "tcp connect error: Connection refused (os error 111)", ... }
```

This typically means the Seafowl engine is not running. Verify that `pgd seafowl_manager` appears in `pg_stat_activity`.

### Error: DDL Global Lock Timeout

```
WARNING: not enough nodes are connected for ddl global lock acquisition
DETAIL:  Missing 1 out of 2 peer nodes required to confirm the ddl global lock.
ERROR:  canceling statement due to global lock timeout
```

This occurs when one or more peer nodes are offline or unreachable. Ensure all PGD nodes are running and reachable before executing DDL statements.

---

## Utility

```sql
-- Check BDR version
SELECT bdr.bdr_version();
```

```bash
# Install additional packages (EDB AS16)
sudo yum install edb-as16-pgfs edb-as16-aidb edb-as16-pgaa
sudo dnf -y install edb-as16-postgres-tuner1
```
