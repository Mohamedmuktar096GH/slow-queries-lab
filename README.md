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
