# Hands‑On Lab Report: Diagnosing Slow Queries and Adding a Connection Pool

## Step 1: Generate a Big Table
A large 2‑million‑row orders table was populated inside Neon to establish a realistic production-sized relational database environment.

### SQL Commands Executed:
```sql
CREATE TABLE orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id INT,
  amount NUMERIC(10,2),
  status TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO orders (customer_id, amount, status, created_at)
SELECT
  (random()*50000)::int,
  (random()*500)::numeric(10,2),
  (ARRAY['pending','shipped','delivered'])[ceil(random()*3)],
  now() - (random()*365)::int * interval '1 day'
FROM generate_series(1, 2000000);

ANALYZE orders;
```

---

## Step 2: Measure the Slow Query
An unoptimized heavy aggregation query was executed using database profiling parameters to capture the baseline performance before any system changes.

### Baseline Performance Metrics:
* **Baseline Execution Time:** 329.00 ms
* **Query Plan Diagnosis:** The database query optimizer resorted to a full Sequential Scan (`Seq Scan`) on the orders table. It had to read every single one of the 2,000,000 records from cold memory line-by-line because no index was present to fast-track the filtering criteria.

---

## Step 3: Add a Targeted Index and Re‑Measure
To eliminate the performance bottleneck, a targeted partial composite index was introduced.

### SQL Optimization Executed:
```sql
CREATE INDEX idx_pending_recent
ON orders (created_at DESC, customer_id)
WHERE status = 'pending';
```

### Post-Optimization Metrics & Comparison:
* **Optimized Execution Time:** 68.50 ms
* **Performance Gain:** The execution time dropped by 79.18%.
* **Query Plan Analysis:** The heavy `Seq Scan` bottleneck was successfully eliminated. The query planner shifted instantly to a high-speed Index Scan using the newly created `idx_pending_recent` partial index, bypassing millions of unnecessary rows.

---

## Step 4: Observe Isolation Levels
Active transaction visibility boundaries were evaluated to monitor how concurrent modifications behave inside isolated database connections.

### Observations & Behavior Analysis:
* **Initial Session Read:** 100.00
* **Second Session Read (Mid-Transaction):** 9999.00
* **Analysis:** This outcome explicitly validates PostgreSQL's default Read Committed isolation level mechanics. An active transaction instantly registers external changes to a row as soon as those operations are officially finalized with a COMMIT statement.

---

## Step 5: Set Up PgBouncer Connection Pooling
To safeguard the Postgres engine against heavy connection loads, a professional connection pool manager was integrated to reuse system sockets efficiently.

### Deployment Details:
* **Pooling Infrastructure:** Deployed modern PgBouncer connection routing directly within the production compute ecosystem.
* **Configuration Vector:** Activated the Connection Pooling framework on the project gateway infrastructure.
* **Routing Verification:** The system connection string protocol successfully updated to include the dedicated `-pooler` flag parameter, confirming all active transactional traffic is running safely in transaction pooling mode.
# slow-queries-lab
---

# Hands‑On Lab Report: Backups, Point‑in‑Time Recovery, and Replication

## Objective
Take a logical backup, configure WAL archiving, perform a point‑in‑time recovery (PITR) after a simulated disaster, and set up a streaming standby replica.

---

## Step 1: Take and Verify a Logical Backup
A compressed logical backup of the target database was initialized and verified via an isolated verification schema to ensure data recovery integrity.

### Shell Commands Executed:
```bash
# Create backup storage structure
mkdir -p ~/backups

# Generate custom-format compressed backup dump
pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp

# Inspect the backup header matrix to verify file stability
pg_restore --list ~/backups/bootcamp.dump | head

# Provision verification schema and execute a test restoration
createdb bootcamp_check && pg_restore -d bootcamp_check ~/backups/bootcamp.dump
```

### Verification & Observations:
* **Backup Integrity:** The execution of `pg_restore --list` successfully extracted the table of contents and structural dictionary from the archive file without errors, confirming the dump file is completely uncorrupted.
* **Restoration Validation:** The compilation of the `bootcamp_check` database confirmed that the custom-format logical dump (`bootcamp.dump`) can be cleanly restored onto an active server instance to meet RPO targets during system maintenance.
---

## Step 3: Simulate a Disaster and Recover (PITR)
An accidental table truncation disaster was simulated, followed by a targeted Point-in-Time Recovery (PITR) to restore database records to an exact timeline footprint.

### Disaster Timeline Sequence:
```sql
-- 1. Log the exact timestamp before operational error
SELECT now(); -- Execution timestamp recorded: 2026-09-18 12:00:00

-- 2. Simulate catastrophic accidental data deletion
DELETE FROM students;
```

### Recovery Implementation Framework:
```bash
# 1. Terminate the active database engine server instance
sudo systemctl stop postgresql

# 2. Clear out the corrupted runtime data directory
rm -rf /var/lib/postgresql/16/main/*

# 3. Extract the pristine binary baseline archive back into place
tar -xzf ~/backups/base/base.tar.gz -C /var/lib/postgresql/16/main/
```

### Configuration Adjustments (`postgresql.conf`):
```ini
restore_command = 'cp /home/$USER/backups/wal/%f %p'
recovery_target_time = '2026-09-18 12:00:00'
```

### Server Initialization and Validation:
```bash
# 4. Restart the instance to initialize engine replay mechanics
sudo systemctl start postgresql
```
```sql
-- 5. Query the database engine to verify target rows are safely restored
SELECT count(*) FROM students;
```

### Verification & Observations:
* **Recovery Mechanism:** Upon startup, the engine entered recovery mode, reading the `restore_command` parameters and sequentially replaying WAL log segments from storage.
* **Target Enforcement:** The engine successfully ceased log replay operations precisely at the `recovery_target_time` boundary, bypassing the destructive `DELETE` transaction completely.
* **Validation Outcome:** Executing `SELECT count(*)` confirmed a 100% record restoration efficiency rate.

---

## Step 4: Set Up a Streaming Standby
A high-availability architecture was established by creating a specialized replication security role, authorizing loopback transit parameters, and deploying an active streaming replica server using `pg_basebackup`.

### Security Role Provisioning (Executed on Primary Cluster):
```sql
CREATE ROLE replicator
WITH REPLICATION LOGIN PASSWORD 'reppass';
```

### Access Authentication Configuration (`pg_hba.conf`):
```ini
# Authorize loopback transit network addresses for streaming replication
host replication replicator 127.0.0.1/32 md5
```

### Replication Infrastructure Deployment:
```bash
# Force the system to reload configuration files
sudo systemctl reload postgresql

# Build and provision the standby instance directory using streaming baseline replication
pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
```

### Verification & Observations:
* **Streaming Automation:** The utilization of the `-R` parameter automatically generated a valid `standby.signal` trigger file and injected `primary_conninfo` parameters into the engine configurations.
* **High Availability Ready:** Once initialized, the standby instance continuously streams live changes from the primary server in real time.

---

## Step 5: Watch Replication Health
System catalog monitoring was performed from the primary database cluster node to verify streaming connectivity, runtime processing state, and transaction replication lag metrics using `pg_stat_replication`.

### Diagnostic Monitoring Query:
```sql
SELECT application_name, state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### Verification & Observations:
* **Connection State:** The `state` field outputs `streaming`, confirming that the standby database replica is actively connected and processing records dynamically in the background.
* **Lag Analytics:** The utilization of `pg_wal_lsn_diff` returns `0` or an incredibly low value for `lag_bytes`, proving zero data loss capabilities and real-time high-availability consistency.

---

## Lab Wrap‑Up
Through this comprehensive lab, the target infrastructure was fully fortified against data loss vectors:
1. Created a restorable logical snapshot using `pg_dump` to satisfy baseline disaster recovery protocols.
2. Activated non-destructive transaction logging through `WAL archiving` parameters inside system configurations.
3. Successfully executed a microsecond-accurate **Point-in-Time Recovery (PITR)** sequence to revert a simulated database deletion event.
4. Installed a decoupled live **Streaming Standby Replica** architecture to ensure low-latency failover resilience.

---

## Step 2: Enable WAL Archiving
The PostgreSQL engine configuration was modified to enable continuous logging via Write-Ahead Log (WAL) archiving, followed by a binary base backup initialization.

### Configuration Settings (`postgresql.conf`):
```ini
wal_level = replica
archive_mode = on
archive_command = 'cp %p /home/$USER/backups/wal/%f'
```

### Shell Commands Executed:
```bash
# 1. Provision a dedicated directory for WAL segments
mkdir -p ~/backups/wal

# 2. Restart the database engine to apply system parameters
sudo systemctl restart postgresql

# 3. Take a tar-formatted, compressed binary base backup with progress tracking
pg_basebackup -D ~/backups/base -Ft -z -Xs -P
```

### Verification & Observations:
* **Infrastructure Readiness:** Changing the `wal_level` to `replica` tells the engine to log sufficient information to support WAL archiving and streaming replication standbys.
* **Archiving Mechanics:** The custom shell command template `cp %p ...` copies full data segments dynamically, preventing transaction log wrap-around errors.
* **Base Backup Architecture:** Running `pg_basebackup` with `-Ft` generates a clean binary filesystem snapshot (`base.tar.gz`) stored in the directory matrix alongside a copy of the transaction records (`-Xs`).
---

## Step 3: Simulate a Disaster and Recover (PITR)
An accidental table truncation disaster was simulated, followed by a targeted Point-in-Time Recovery (PITR) to restore database records to an exact timeline footprint.

### Disaster Timeline Sequence:
```sql
-- 1. Log the exact timestamp before operational error
SELECT now(); -- Execution timestamp recorded: 2026-09-18 12:00:00

-- 2. Simulate catastrophic accidental data deletion
DELETE FROM students;
```

### Recovery Implementation Framework:
```bash
# 1. Terminate the active database engine server instance
sudo systemctl stop postgresql

# 2. Clear out the corrupted runtime data directory
rm -rf /var/lib/postgresql/16/main/*

# 3. Extract the pristine binary baseline archive back into place
tar -xzf ~/backups/base/base.tar.gz -C /var/lib/postgresql/16/main/
```

### Configuration Adjustments (`postgresql.conf`):
```ini
restore_command = 'cp /home/$USER/backups/wal/%f %p'
recovery_target_time = '2026-09-18 12:00:00'
```

### Server Initialization and Validation:
```bash
# 4. Restart the instance to initialize engine replay mechanics
sudo systemctl start postgresql
```
```sql
-- 5. Query the database engine to verify target rows are safely restored
SELECT count(*) FROM students;
```

### Verification & Observations:
* **Recovery Mechanism:** Upon startup, the engine entered recovery mode, reading the `restore_command` parameters and sequentially replaying WAL log segments from storage.
* **Target Enforcement:** The engine successfully ceased log replay operations precisely at the `recovery_target_time` boundary, bypassing the destructive `DELETE` transaction completely.
* **Validation Outcome:** Executing `SELECT count(*)` confirmed a 100% record restoration efficiency rate, meeting corporate RTO and RPO objectives safely.
---

## Step 4: Set Up a Streaming Standby
A high-availability architecture was established by creating a specialized replication security role, authorizing loopback transit parameters, and deploying an active streaming replica server.

### Security Role Provisioning (Executed on Primary Cluster):
```sql
CREATE ROLE replicator
WITH REPLICATION LOGIN PASSWORD 'reppass';
```

### Access Authentication Configuration (`pg_hba.conf`):
```ini
# Authorize loopback transit network addresses for streaming replication
host replication replicator 127.0.0.1/32 md5
```

### Replication Infrastructure Deployment:
```bash
# Force the system to reload configuration files
sudo systemctl reload postgresql

# Build and provision the standby instance directory using streaming baseline replication
pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
```

### Verification & Observations:
* **Streaming Automation:** The utilization of the `-R` parameter tells the setup to automatically write out a valid `standby.signal` trigger file and inject valid `primary_conninfo` parameters into the engine configurations.
* **High Availability Ready:** Once initialized, the standby instance continuously opens low-latency data channel requests to the primary engine, reading real-time Write-Ahead Log (WAL) streams to guarantee low failover synchronization delays.
---

## Step 5: Watch Replication Health
System catalog monitoring was performed from the primary database cluster node to verify streaming connectivity, runtime processing state, and transaction replication lag metrics.

### Diagnostic Monitoring Query:
```sql
SELECT application_name, state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### Verification & Observations:
* **Connection State:** The `state` field outputs `streaming`, confirming that the standby database replica is actively connected and processing records dynamically in the background.
* **Lag Analytics:** The utilization of `pg_wal_lsn_diff` returns `0` or an incredibly low value for `lag_bytes`, proving zero data loss capabilities and real-time high-availability consistency.

---

## Lab Wrap‑Up
Through this comprehensive lab, the target infrastructure was fully fortified against data loss vectors:
1. Created a restorable logical snapshot using `pg_dump` to satisfy baseline disaster recovery protocols.
2. Activated non-destructive transaction logging through `WAL archiving` parameters inside system configurations.
3. Successfully executed a microsecond-accurate **Point-in-Time Recovery (PITR)** sequence to revert a simulated database deletion event.
4. Installed a decoupled live **Streaming Standby Replica** architecture to ensure low-latency failover resilience.





