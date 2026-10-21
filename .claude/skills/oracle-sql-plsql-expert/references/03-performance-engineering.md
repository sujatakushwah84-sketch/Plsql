# Oracle Performance Engineering (HIGH PRIORITY)

## SQL Tuning Workflow

1. **Identify** the problem SQL — AWR "SQL ordered by Elapsed Time",
   `V$SQL`, ASH, or the user-reported screen/job.
2. **Measure** the real plan with runtime statistics (not just `EXPLAIN PLAN`).
3. **Find the biggest gap** — the step where estimated rows (`E-Rows`) differ
   most from actual rows (`A-Rows`), or where most time/buffers accumulate.
4. **Fix the root cause** — statistics, predicate rewrite, index, partition
   pruning, join order, or design — in that order of preference.
5. **Verify** with the same measurement; record before/after (elapsed, buffer
   gets, rows). Stabilize with a SQL Plan Baseline if needed.

```sql
-- Find top SQL by elapsed time
SELECT sql_id, plan_hash_value, executions,
       ROUND(elapsed_time / 1e6, 2)                          AS elapsed_s,
       ROUND(elapsed_time / NULLIF(executions, 0) / 1e3, 2)  AS ms_per_exec,
       buffer_gets, disk_reads, rows_processed,
       SUBSTR(sql_text, 1, 100)                              AS sql_text
FROM   v$sql
ORDER  BY elapsed_time DESC
FETCH  FIRST 20 ROWS ONLY;
```

## Explain Plan & Execution Plan Analysis

```sql
-- Estimated plan (may differ from the real one: no bind peeking, no adaptive info)
EXPLAIN PLAN FOR
SELECT * FROM orders WHERE customer_id = :cust;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(format => 'TYPICAL +PREDICATE'));

-- Actual plan with runtime row counts
SELECT /*+ GATHER_PLAN_STATISTICS */ * FROM orders WHERE customer_id = 42;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PEEKED_BINDS +OUTLINE'));

-- Plan of a cursor already in the shared pool / in AWR
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', NULL, 'ALLSTATS LAST'));
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_AWR('&sql_id'));
```

How to read a plan:

- Execution starts at the most-indented leaf; children of a join run first,
  the first child drives.
- Key columns: `Starts`, `E-Rows`, `A-Rows`, `A-Time`, `Buffers`, `Reads`,
  `OMem/Used-Mem` (sort/hash spill to temp = `1-pass`/`multi-pass`).
- **E-Rows × Starts vs. A-Rows** off by orders of magnitude = cardinality
  misestimate → wrong join method/order. This is the #1 root cause.
- Check the **Predicate Information** section: `access` (uses index/partition)
  vs. `filter` (applied after reading rows); implicit conversions such as
  `INTERNAL_FUNCTION` or `TO_NUMBER(col)` disable indexes.

Common access paths and join methods:

| Operation | Good when | Red flag when |
|-----------|-----------|---------------|
| `TABLE ACCESS FULL` | Reading a large fraction of rows; multiblock reads | Returns few rows from a big table |
| `INDEX UNIQUE / RANGE SCAN` | Selective predicate | Range scan + `TABLE ACCESS BY ROWID` for millions of rows |
| `INDEX FAST FULL SCAN` | Index covers all columns | — |
| `INDEX SKIP SCAN` | Leading column has few distinct values | Often indicates a missing better index |
| `NESTED LOOPS` | Small driving set, indexed inner | Driving set is large (high `Starts`) |
| `HASH JOIN` | Large unsorted sets, equality join | Build side spills to temp |
| `MERGE JOIN` | Inputs already sorted / non-equi joins | Large sorts |
| `FILTER` with subquery | Few outer rows | Runs subquery millions of times |

## Oracle Optimizer

- Cost-Based Optimizer (CBO) estimates cost from statistics: table rows/blocks,
  column NDV, nulls, histograms, index clustering factor, system stats.
- Query transformations: view merging, subquery unnesting, predicate pushing,
  join elimination, OR-expansion, star transformation.
- **Bind peeking & Adaptive Cursor Sharing**: a plan peeked for one bind value
  may be bad for another on skewed data; ACS marks cursors bind-sensitive /
  bind-aware (`V$SQL.IS_BIND_SENSITIVE`, `IS_BIND_AWARE`).
- **Adaptive plans (12c+)**: join method decided at runtime
  (`+ADAPTIVE` format in DISPLAY_CURSOR).
- Useful parameters (session-level for testing): `OPTIMIZER_MODE`
  (`ALL_ROWS` / `FIRST_ROWS_n`), `OPTIMIZER_FEATURES_ENABLE`,
  `OPTIMIZER_ADAPTIVE_PLANS`.
- Hints are a last resort and a diagnostic tool: `LEADING`, `USE_NL`, `USE_HASH`,
  `INDEX`, `FULL`, `PARALLEL`, `CARDINALITY`, `NO_MERGE`, `UNNEST`,
  `PUSH_PRED`. Prefer fixing statistics or SQL; if hints are needed in
  production, wrap them in a SQL Patch or Baseline rather than editing app code.

## Plan Stability

```sql
-- Load a known good plan from the cursor cache into a SQL Plan Baseline
DECLARE
    l_cnt PLS_INTEGER;
BEGIN
    l_cnt := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
                 sql_id          => '&sql_id',
                 plan_hash_value => &good_plan_hash);
END;
/
```

- SQL Plan Management (baselines), SQL Profiles (from SQL Tuning Advisor),
  SQL Patches (`DBMS_SQLDIAG.CREATE_SQL_PATCH`) to inject hints without code change.
- After an upgrade or stats refresh, compare `plan_hash_value` history in
  `DBA_HIST_SQLSTAT` to spot plan regressions.

## Index Optimization

- Index columns used in selective `WHERE`, `JOIN`, and `ORDER BY` clauses;
  always index foreign key columns (prevents table locks on parent
  DELETE/UPDATE and speeds joins).
- B-tree for OLTP; bitmap for low-cardinality columns in read-mostly DW (never
  on heavily updated OLTP tables — lock contention).
- **Clustering factor** close to table blocks = efficient range scans; close to
  rows = the optimizer will prefer full scans.
- Covering indexes avoid `TABLE ACCESS BY ROWID`.
- Index-killers: functions on the column (`TRUNC(order_date) = ...`), implicit
  conversion (`varchar_col = 123`), leading wildcard (`LIKE '%abc'`),
  `col != value`, `col IS NULL` (B-tree does not store all-null keys — add a
  constant: `CREATE INDEX ix ON t (col, 0)`).
- Cost of indexes: slower DML, more redo, more space. Find unused indexes with
  `ALTER INDEX ... MONITORING USAGE` / `DBA_INDEX_USAGE` (12.2+); test drops with
  `ALTER INDEX ... INVISIBLE`.

### Composite & Function-Based Indexes

```sql
-- Composite: equality columns first, then range column
CREATE INDEX ix_orders_cust_status_dt ON orders (customer_id, status, order_date);
-- Serves: WHERE customer_id = :c AND status = :s AND order_date >= :d
--         WHERE customer_id = :c                       (leading column)
-- Not efficiently: WHERE status = :s                   (skip scan at best)

-- Function-based
CREATE INDEX ix_cust_upper_email ON customers (UPPER(email));
SELECT * FROM customers WHERE UPPER(email) = UPPER(:email);

-- Index only the "interesting" rows (NULLs are not indexed)
CREATE INDEX ix_orders_pending ON orders (CASE WHEN status = 'PENDING' THEN order_id END);
SELECT * FROM orders WHERE CASE WHEN status = 'PENDING' THEN order_id END IS NOT NULL;
```

- Column order: equality predicates → range predicates → columns for
  `ORDER BY` / covering. Higher selectivity first only matters among equality columns.
- Gather stats after creating a function-based index (hidden virtual column).
- Compression (`COMPRESS n`, advanced index compression) for repeated prefixes.

## Statistics

```sql
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname          => 'APP',
        tabname          => 'ORDERS',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
        cascade          => TRUE,
        degree           => 4,
        no_invalidate    => DBMS_STATS.AUTO_INVALIDATE);
END;
/

-- Column group for correlated columns (fixes underestimates on combined predicates)
SELECT DBMS_STATS.CREATE_EXTENDED_STATS('APP', 'ORDERS', '(COUNTRY, STATE)') FROM dual;

-- Check staleness
SELECT table_name, last_analyzed, num_rows, stale_stats
FROM   dba_tab_statistics WHERE owner = 'APP' AND stale_stats = 'YES';
```

- Histograms (frequency, top-frequency, hybrid) capture skew.
- After bulk loads in batch jobs, gather stats before dependent queries run;
  use incremental stats on partitioned tables; lock stats for volatile staging
  tables (or set representative stats) to prevent bad plans.
- Back up before changing: `DBMS_STATS.EXPORT_TABLE_STATS`; history and
  `RESTORE_TABLE_STATS` allow reverting a regression.
- Dynamic sampling (`OPTIMIZER_DYNAMIC_SAMPLING`) helps with GTTs and complex predicates.

## Partitioning

```sql
CREATE TABLE sales (
    sale_id     NUMBER        NOT NULL,
    sale_date   DATE          NOT NULL,
    region      VARCHAR2(10)  NOT NULL,
    amount      NUMBER(12,2)
)
PARTITION BY RANGE (sale_date)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
SUBPARTITION BY LIST (region)
SUBPARTITION TEMPLATE (
    SUBPARTITION sp_na   VALUES ('NA'),
    SUBPARTITION sp_emea VALUES ('EMEA'),
    SUBPARTITION sp_oth  VALUES (DEFAULT)
)
(PARTITION p_initial VALUES LESS THAN (DATE '2024-01-01'));

CREATE INDEX ix_sales_region ON sales (region, sale_date) LOCAL;
```

- Strategies: range/interval (time series), list (discrete values), hash
  (even distribution, reduce hot blocks), composite, reference, auto-list (12.2+).
- **Partition pruning** is the primary performance benefit — verify
  `Pstart/Pstop` in the plan; functions on the partition key or type
  mismatches prevent pruning.
- **Partition-wise joins** when two tables are equi-partitioned on the join key.
- Lifecycle: `TRUNCATE` / `DROP PARTITION ... UPDATE INDEXES` for purging,
  `EXCHANGE PARTITION` for near-instant loads, `MOVE PARTITION ONLINE ... COMPRESS`
  for archiving older data.
- Local indexes = easy maintenance; global indexes = good for non-partition-key
  lookups but need `UPDATE GLOBAL INDEXES` on partition DDL.

## Bind Variables

```sql
-- Bad: hard parse per value, shared pool flooding, SQL injection risk
EXECUTE IMMEDIATE 'SELECT name FROM customers WHERE id = ' || l_id INTO l_name;

-- Good
EXECUTE IMMEDIATE 'SELECT name FROM customers WHERE id = :id' INTO l_name USING l_id;
```

- Static SQL in PL/SQL binds automatically.
- Detect non-binding apps: many `V$SQL` rows with the same `FORCE_MATCHING_SIGNATURE`,
  high "parse count (hard)", latch/mutex waits on the shared pool / library cache.
- `CURSOR_SHARING = FORCE` is a stopgap, not a fix.
- For skewed columns, binds plus histograms rely on adaptive cursor sharing;
  sometimes literals are deliberately better for a few very skewed values.

## Query Rewrite

Rewrite patterns that frequently give large wins:

| Anti-pattern | Rewrite |
|--------------|---------|
| Row-by-row loop with DML | Single `INSERT ... SELECT` / `MERGE` / `UPDATE` |
| `WHERE TRUNC(dt) = :d` | `WHERE dt >= :d AND dt < :d + 1` |
| `NOT IN (subquery)` with nullable column | `NOT EXISTS` |
| Correlated scalar subqueries per row | Join or analytic function |
| Multiple passes over the same table | One pass with conditional aggregation |
| `SELECT DISTINCT` to hide join duplicates | Fix the join / use `EXISTS` |
| `OR` across different columns | `UNION ALL` of selective branches |
| `UNION` where duplicates are impossible | `UNION ALL` |
| Function call per row in SQL | Join to the data or use scalar subquery caching |
| `COUNT(*) > 0` to test existence | `EXISTS` / `FETCH FIRST 1 ROW ONLY` |

**Materialized view query rewrite**: with `ENABLE QUERY REWRITE` and
`QUERY_REWRITE_ENABLED = TRUE`, the optimizer transparently redirects
aggregate queries to a matching MV.

## Performance Bottleneck Analysis

```sql
-- What is the database waiting on right now?
SELECT event, wait_class, COUNT(*) AS sessions
FROM   v$session
WHERE  status = 'ACTIVE' AND wait_class <> 'Idle'
GROUP  BY event, wait_class
ORDER  BY sessions DESC;

-- Top SQL and events over the last 30 minutes (ASH — requires Diagnostics Pack)
SELECT sql_id, event, COUNT(*) AS samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '30' MINUTE
GROUP  BY sql_id, event
ORDER  BY samples DESC
FETCH  FIRST 20 ROWS ONLY;
```

Common wait events and what they point to:

| Wait event | Likely cause |
|------------|--------------|
| `db file sequential read` | Single-block I/O — index lookups, often NL joins over many rows |
| `db file scattered read` / `direct path read` | Full scans |
| `enq: TX - row lock contention` | Application row locking / uncommitted transactions |
| `enq: TM - contention` | Unindexed foreign keys |
| `log file sync` | Too-frequent commits (commit in loops) or slow redo I/O |
| `latch: shared pool` / `library cache` / `cursor: pin S wait on X` | Hard parsing, no binds |
| `direct path write temp` | Sorts/hash joins spilling to temp — PGA or plan problem |
| `gc buffer busy` (RAC) | Hot blocks across instances |
| `CPU` | Inefficient SQL (high logical I/O), PL/SQL compute |

Tools: AWR/ADDM reports (`awrrpt.sql`, `addmrpt.sql`), ASH report,
SQL Monitor (`DBMS_SQLTUNE.REPORT_SQL_MONITOR`) for long-running or parallel
SQL, `DBMS_SQLTUNE` (SQL Tuning Advisor), 10046 trace + `tkprof` for exact
per-statement call breakdown.

## High-Volume Data Processing

- Set-based SQL first: `INSERT /*+ APPEND */ INTO ... SELECT` (direct path,
  minimal undo), `CREATE TABLE AS SELECT`, `MERGE`.
- Parallel: `ALTER SESSION ENABLE PARALLEL DML;` + `/*+ PARALLEL(n) */`.
- Replacing a massive `UPDATE`/`DELETE` with CTAS + rename or partition
  exchange is often orders of magnitude faster.
- `DBMS_PARALLEL_EXECUTE` to chunk work by rowid / number range with
  restartability.
- PL/SQL: BULK COLLECT `LIMIT` + FORALL `SAVE EXCEPTIONS`; DML error logging
  (`LOG ERRORS INTO`) for set-based loads.
- Commit by business unit of work, not per row; design restartable batches.
