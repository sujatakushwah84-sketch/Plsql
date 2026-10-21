# Production & Troubleshooting

## Incident Resolution Workflow

1. **Triage** — scope (one user, one module, everything?), severity, start time,
   recent changes (deployments, stats jobs, data loads, parameter changes).
2. **Stabilize** — restore service first: kill a runaway session, pin a known
   good plan, disable a failing job, add a temporary resource. Document every action.
3. **Diagnose** — gather evidence before it ages out of `V$` views/ASH:
   AWR/ASH snapshots, alert log, trace files, error logs, application logs.
4. **Fix** — minimal, tested, reversible change through the change process.
5. **Root cause analysis & prevention** — write it up, add monitoring/tests.

## Production Issue Debugging — Toolkit

```sql
-- What is a session doing right now?
SELECT s.sid, s.serial#, s.username, s.status, s.module, s.action,
       s.sql_id, s.event, s.wait_class, s.seconds_in_wait,
       s.blocking_session, s.last_call_et
FROM   v$session s
WHERE  s.username IS NOT NULL
AND    s.status = 'ACTIVE';

-- Long-running operations progress
SELECT sid, opname, target, sofar, totalwork,
       ROUND(sofar / NULLIF(totalwork, 0) * 100, 1) AS pct_done,
       time_remaining
FROM   v$session_longops
WHERE  sofar < totalwork;

-- Full text of a SQL
SELECT sql_fulltext FROM v$sql WHERE sql_id = '&sql_id' AND ROWNUM = 1;

-- Kill (with approval)
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```

- Tracing a session: `DBMS_MONITOR.SESSION_TRACE_ENABLE(sid, serial#, waits => TRUE, binds => TRUE)`
  then `tkprof` the trace file.
- Reproduce with the same binds (`V$SQL_BIND_CAPTURE`) and the same optimizer
  environment — not with literals in SQL Developer.
- Check invalid objects after deployments:
  `SELECT owner, object_name, object_type FROM dba_objects WHERE status = 'INVALID';`
  and recompile with `UTL_RECOMP` / `DBMS_UTILITY.COMPILE_SCHEMA`.

## Database Performance Troubleshooting

"It was fast yesterday" checklist:

1. Did the **plan change**? Compare `plan_hash_value` in
   `DBA_HIST_SQLSTAT` over time.

```sql
SELECT sn.begin_interval_time, st.plan_hash_value, st.executions_delta,
       ROUND(st.elapsed_time_delta / NULLIF(st.executions_delta, 0) / 1e3, 1) AS ms_per_exec,
       ROUND(st.buffer_gets_delta  / NULLIF(st.executions_delta, 0))          AS gets_per_exec
FROM   dba_hist_sqlstat st
       JOIN dba_hist_snapshot sn ON sn.snap_id = st.snap_id
                                AND sn.instance_number = st.instance_number
WHERE  st.sql_id = '&sql_id'
ORDER  BY sn.begin_interval_time;
```

2. Did **data volume or distribution** change (a load, purge failure, skew)?
3. Did **statistics** change (stale, missing, gathered at a bad moment on an
   empty staging table)?
4. Is it **contention** (locks, latches) rather than the SQL itself?
5. Is the **system** saturated (CPU, I/O, memory, temp, redo)?
6. Did the **application** change (new binds, new call pattern, connection pool size)?

Fast remedies for a plan regression: load the previous good plan into a SQL
Plan Baseline, restore previous statistics, or apply a SQL Patch — then
investigate the underlying cause.

## Error Analysis — Common ORA- Errors

| Error | Meaning | Typical root cause / fix |
|-------|---------|--------------------------|
| ORA-00001 | Unique constraint violated | Duplicate data, sequence behind max(id), retry logic |
| ORA-00054 | Resource busy, NOWAIT | Locked object during DDL — use `DDL_LOCK_TIMEOUT` or a quiet window |
| ORA-00060 | Deadlock detected | Inconsistent lock order, unindexed FKs, bitmap indexes on OLTP |
| ORA-01000 | Max open cursors exceeded | Cursor leak — unclosed cursors / ResultSets in app code |
| ORA-01403 | No data found | `SELECT INTO` with no rows — handle or use aggregate/outer join |
| ORA-01422 | Exact fetch returns more than requested rows | `SELECT INTO` returns >1 row — data or predicate issue |
| ORA-01427 | Single-row subquery returns more than one row | Duplicate data in lookup |
| ORA-01438 / ORA-12899 | Value too large for column | Data length/precision; multi-byte chars (`BYTE` vs. `CHAR`) |
| ORA-01476 | Divisor is zero | `NULLIF(divisor, 0)` |
| ORA-01555 | Snapshot too old | Long query + undo overwritten — tune query, undo retention, avoid fetch-across-commit |
| ORA-01652 | Unable to extend temp segment | Huge sort/hash join — usually a bad plan or cartesian join |
| ORA-01722 / ORA-01858 | Invalid number / date | Implicit conversion on dirty data; predicate order not guaranteed |
| ORA-04030 / ORA-04031 | Out of process / shared memory | Unbounded collections (missing `LIMIT`), no binds |
| ORA-04068 | Package state discarded | Redeploy while sessions hold state |
| ORA-04091 | Mutating table | Row trigger querying its own table — compound trigger |
| ORA-06502 | Numeric or value error | Variable too small, conversion failure |
| ORA-06512 | At line … | Stack location — read the error *above* it |
| ORA-06519 | Active autonomous transaction rolled back | Missing COMMIT in autonomous block |
| ORA-30926 | Unable to get stable set of rows | Duplicate keys in MERGE source |
| ORA-00942 | Table or view does not exist | Missing grant/synonym, wrong schema |

Always read the full stack (`FORMAT_ERROR_STACK` + `FORMAT_ERROR_BACKTRACE`):
the first error is the cause, the rest is where it travelled.

## Data Validation & Data Integrity

```sql
-- Duplicates on a business key
SELECT order_no, COUNT(*)
FROM   orders
GROUP  BY order_no
HAVING COUNT(*) > 1;

-- Orphans (FK not enforced, e.g. legacy or staging)
SELECT oi.order_id
FROM   order_items oi
WHERE  NOT EXISTS (SELECT 1 FROM orders o WHERE o.order_id = oi.order_id);

-- Totals reconciliation between header and lines
SELECT o.order_id, o.total, SUM(oi.quantity * oi.unit_price) AS line_total
FROM   orders o JOIN order_items oi ON oi.order_id = o.order_id
GROUP  BY o.order_id, o.total
HAVING o.total <> SUM(oi.quantity * oi.unit_price);

-- Source vs. target row counts and checksums
SELECT COUNT(*), SUM(ORA_HASH(id || '|' || amount || '|' || status)) FROM src_table;
SELECT COUNT(*), SUM(ORA_HASH(id || '|' || amount || '|' || status)) FROM tgt_table;
```

- Prevent first: constraints, NOT NULL, CHECK, FKs, unique indexes.
- Validate loads: row counts, control totals, rejects via `LOG ERRORS INTO`,
  `MINUS` reconciliation, and invalid data reports (`VALIDATE_CONVERSION`, 12.2+).
- Data fixes in production: script with `WHERE` clauses verified by a prior
  `SELECT COUNT(*)`, back up affected rows (`CREATE TABLE bkp_... AS SELECT`),
  expected-row-count assertions, peer review, and a rollback script.
- Recovery options: Flashback Query (`AS OF TIMESTAMP`), Flashback Table,
  Flashback Drop (recycle bin), LogMiner for "who changed what".

```sql
-- See the data as it was 30 minutes ago
SELECT * FROM orders AS OF TIMESTAMP SYSTIMESTAMP - INTERVAL '30' MINUTE
WHERE  order_id = 1001;
```

## Root Cause Analysis

Structure every RCA:

1. **Summary** — what happened, impact (users, duration, data), severity.
2. **Timeline** — detection, escalation, mitigation, resolution (with timestamps).
3. **Root cause** — use "5 Whys"; distinguish trigger (what started it) from
   root cause (why the system allowed it).
4. **Evidence** — AWR/ASH extracts, plans, error stacks, log lines.
5. **Resolution** — immediate fix and permanent fix.
6. **Prevention** — monitoring, alerts, tests, code standards, process change.

Example: *Trigger*: nightly stats job ran while staging table was empty.
*Root cause*: volatile table had no locked/representative statistics, so the
optimizer chose a nested loop over 20M rows. *Prevention*: lock stats on
staging tables, gather after load in the batch, baseline the critical SQL.

## Deployment & Release Support

- **Scripts**: versioned in source control, re-runnable (idempotent checks
  against `USER_TABLES` / `USER_TAB_COLUMNS`), ordered (DDL → code → data →
  grants → recompile), with a matching rollback script.
- **Tooling**: Liquibase / Flyway / SQLcl `liquibase` for schema migration.
- **Pre-release**: code review, unit tests passed, explain plans for new or
  changed SQL on production-like volumes, impact analysis via `DBA_DEPENDENCIES`.
- **Low-impact DDL**: `ONLINE` operations (`CREATE INDEX ... ONLINE`,
  `ALTER TABLE ... MOVE ONLINE`), `DDL_LOCK_TIMEOUT`, invisible indexes, adding
  nullable columns / columns with defaults (metadata-only in 11g+/12c+),
  `DBMS_REDEFINITION` for online table restructuring, Edition-Based
  Redefinition for code.
- **Post-release**: invalid object check, smoke tests, watch AWR/top SQL and
  error logs, keep the rollback window open.

```sql
-- Idempotent column add
DECLARE
    l_cnt PLS_INTEGER;
BEGIN
    SELECT COUNT(*) INTO l_cnt
    FROM   user_tab_columns
    WHERE  table_name = 'ORDERS' AND column_name = 'CHANNEL';

    IF l_cnt = 0 THEN
        EXECUTE IMMEDIATE 'ALTER TABLE orders ADD (channel VARCHAR2(20) DEFAULT ''WEB'' NOT NULL)';
    END IF;
END;
/
```
