# Core & Advanced Oracle SQL

## Complex Joins

Use ANSI join syntax for readability; keep the old `(+)` syntax only when
maintaining legacy code.

```sql
-- Inner, outer, and anti/semi join in one query
SELECT d.department_name,
       e.employee_id,
       e.last_name,
       m.last_name AS manager_name
FROM   departments d
       LEFT  JOIN employees e ON e.department_id = d.department_id
       LEFT  JOIN employees m ON m.employee_id   = e.manager_id
WHERE  NOT EXISTS (SELECT 1
                   FROM   job_history jh
                   WHERE  jh.employee_id = e.employee_id);
```

Patterns and pitfalls:

- **Outer-join filter placement**: a predicate on the outer table in `WHERE`
  turns a LEFT JOIN into an inner join. Put it in the `ON` clause instead.
- **Self joins** for hierarchies/comparisons (employee → manager).
- **Semi join** (`EXISTS` / `IN`) returns each driving row once; **anti join**
  (`NOT EXISTS`) — prefer it over `NOT IN`, which returns nothing if the
  subquery yields a `NULL`.
- **`CROSS APPLY` / `OUTER APPLY` / `LATERAL`** (12c+) for "top-N per row".
- **Partitioned outer join** for densifying sparse data:

```sql
SELECT p.product_id, c.cal_date, NVL(s.qty, 0) AS qty
FROM   sales s PARTITION BY (s.product_id)
       RIGHT OUTER JOIN calendar c ON c.cal_date = s.sale_date;
```

## Subqueries & Correlated Subqueries

```sql
-- Scalar subquery
SELECT e.last_name,
       (SELECT d.department_name
        FROM   departments d
        WHERE  d.department_id = e.department_id) AS dept
FROM   employees e;

-- Correlated: employees earning above their department average
SELECT e.employee_id, e.salary
FROM   employees e
WHERE  e.salary > (SELECT AVG(x.salary)
                   FROM   employees x
                   WHERE  x.department_id = e.department_id);
```

- A correlated subquery executes logically per outer row; the optimizer may
  unnest it into a join. Check the plan for `FILTER` operations — a `FILTER`
  driving a large subquery is often the bottleneck.
- The analytic rewrite usually reads the table once:

```sql
SELECT employee_id, salary
FROM  (SELECT e.*, AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
       FROM   employees e)
WHERE  salary > dept_avg;
```

## Common Table Expressions (WITH clause)

```sql
WITH dept_totals AS (
    SELECT department_id, SUM(salary) AS total_sal
    FROM   employees
    GROUP  BY department_id
),
company AS (
    SELECT AVG(total_sal) AS avg_sal FROM dept_totals
)
SELECT d.department_id, d.total_sal
FROM   dept_totals d CROSS JOIN company c
WHERE  d.total_sal > c.avg_sal;
```

- `/*+ MATERIALIZE */` forces a temp table; `/*+ INLINE */` forces merging.
  Oracle materializes automatically when a CTE is referenced more than once.
- **Recursive CTE** (11gR2+):

```sql
WITH org (employee_id, manager_id, lvl, path) AS (
    SELECT employee_id, manager_id, 1, CAST(last_name AS VARCHAR2(4000))
    FROM   employees
    WHERE  manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.manager_id, o.lvl + 1, o.path || ' > ' || e.last_name
    FROM   employees e
           JOIN org o ON e.manager_id = o.employee_id
)
SEARCH DEPTH FIRST BY employee_id SET order_seq
CYCLE employee_id SET is_cycle TO 'Y' DEFAULT 'N'
SELECT * FROM org ORDER BY order_seq;
```

- `WITH FUNCTION` (12c+) defines inline PL/SQL functions without context
  switching overhead from a stored function.

## Analytical (Window) Functions

```sql
SELECT employee_id,
       department_id,
       salary,
       ROW_NUMBER()   OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn,
       RANK()         OVER (PARTITION BY department_id ORDER BY salary DESC) AS rnk,
       DENSE_RANK()   OVER (PARTITION BY department_id ORDER BY salary DESC) AS drnk,
       LAG(salary)    OVER (PARTITION BY department_id ORDER BY hire_date)   AS prev_sal,
       LEAD(salary)   OVER (PARTITION BY department_id ORDER BY hire_date)   AS next_sal,
       SUM(salary)    OVER (PARTITION BY department_id ORDER BY hire_date
                            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
       AVG(salary)    OVER (ORDER BY hire_date
                            RANGE BETWEEN INTERVAL '90' DAY PRECEDING AND CURRENT ROW) AS avg_90d,
       RATIO_TO_REPORT(salary) OVER (PARTITION BY department_id) AS pct_of_dept,
       FIRST_VALUE(last_name)  OVER (PARTITION BY department_id ORDER BY salary DESC) AS top_earner,
       NTILE(4)       OVER (ORDER BY salary) AS quartile
FROM   employees;
```

- Default window with `ORDER BY` is `RANGE UNBOUNDED PRECEDING` — ties are
  included together. Use `ROWS` for strict running totals.
- `LAST_VALUE` needs `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.
- `KEEP (DENSE_RANK FIRST/LAST ORDER BY ...)` returns a value from the first/last
  row of a group without a self join.
- `LISTAGG(... ON OVERFLOW TRUNCATE)` (12cR2+), `LISTAGG(DISTINCT ...)` (19c+).

**Gaps and islands** — group consecutive dates:

```sql
SELECT account_id, MIN(txn_date) AS start_dt, MAX(txn_date) AS end_dt
FROM  (SELECT account_id, txn_date,
              txn_date - ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY txn_date) AS grp
       FROM   daily_activity)
GROUP  BY account_id, grp;
```

**MATCH_RECOGNIZE** (12c+) for row-pattern matching (e.g. V-shaped price moves):

```sql
SELECT *
FROM   stock_ticks
MATCH_RECOGNIZE (
    PARTITION BY symbol ORDER BY tick_time
    MEASURES STRT.tick_time AS start_time, LAST(DOWN.tick_time) AS bottom_time
    ONE ROW PER MATCH
    PATTERN (STRT DOWN+ UP+)
    DEFINE DOWN AS price < PREV(price),
           UP   AS price > PREV(price)
);
```

## Hierarchical Queries

```sql
SELECT LPAD(' ', 2 * (LEVEL - 1)) || last_name AS org_chart,
       LEVEL,
       SYS_CONNECT_BY_PATH(last_name, '/') AS path,
       CONNECT_BY_ROOT last_name           AS top_manager,
       CONNECT_BY_ISLEAF                   AS is_leaf
FROM   employees
START  WITH manager_id IS NULL
CONNECT BY NOCYCLE PRIOR employee_id = manager_id
ORDER  SIBLINGS BY last_name;
```

- `WHERE` filters rows *after* the hierarchy is built; filters in
  `CONNECT BY` prune branches during traversal.
- Generate rows: `SELECT LEVEL FROM dual CONNECT BY LEVEL <= 100`.
- Index both `employee_id` and `manager_id` for large trees.

## MERGE / UPSERT

```sql
MERGE INTO customers tgt
USING (SELECT customer_id, name, email, status
       FROM   stg_customers) src
ON    (tgt.customer_id = src.customer_id)
WHEN MATCHED THEN
    UPDATE SET tgt.name       = src.name,
               tgt.email      = src.email,
               tgt.updated_at = SYSTIMESTAMP
    WHERE  DECODE(tgt.name,  src.name,  0, 1) = 1
       OR  DECODE(tgt.email, src.email, 0, 1) = 1     -- update only real changes (NULL-safe)
    DELETE WHERE src.status = 'CLOSED'
WHEN NOT MATCHED THEN
    INSERT (customer_id, name, email, created_at)
    VALUES (src.customer_id, src.name, src.email, SYSTIMESTAMP)
    WHERE  src.status <> 'CLOSED';
```

- Source must be deterministic per target row, otherwise **ORA-30926**
  (unable to get a stable set of rows) — deduplicate the source first.
- Cannot update columns used in the `ON` clause (ORA-38104).
- Combine with `LOG ERRORS INTO err$_customers REJECT LIMIT UNLIMITED` for
  bulk loads that must not fail entirely on bad rows.

## Set Operators

| Operator | Result | Notes |
|----------|--------|-------|
| `UNION ALL` | All rows | No sort; fastest — default choice |
| `UNION` | Distinct rows | Adds a sort/hash for dedup |
| `INTERSECT` | Rows in both | Distinct |
| `MINUS` / `EXCEPT` (21c+) | Rows in first not in second | Distinct; great for data reconciliation |

```sql
-- Reconcile source vs. target in both directions
(SELECT 'MISSING_IN_TGT' src, id, amount FROM src_table
 MINUS
 SELECT 'MISSING_IN_TGT', id, amount FROM tgt_table)
UNION ALL
(SELECT 'EXTRA_IN_TGT', id, amount FROM tgt_table
 MINUS
 SELECT 'EXTRA_IN_TGT', id, amount FROM src_table);
```

Columns are matched by position; datatypes must be compatible; `ORDER BY`
applies only to the final result. 21c adds `MINUS ALL` / `INTERSECT ALL`,
which keep duplicates.

## Advanced Aggregation

```sql
SELECT region, product, channel,
       SUM(amount)  AS total,
       GROUPING_ID(region, product, channel) AS gid
FROM   sales
GROUP  BY GROUPING SETS ((region, product, channel),
                         (region, product),
                         (region),
                         ());

-- ROLLUP: hierarchical subtotals; CUBE: all combinations
SELECT DECODE(GROUPING(region), 1, 'ALL REGIONS', region) AS region,
       product, SUM(amount)
FROM   sales
GROUP  BY ROLLUP (region, product);
```

**PIVOT / UNPIVOT**

```sql
SELECT *
FROM  (SELECT region, quarter, amount FROM sales)
PIVOT (SUM(amount) FOR quarter IN ('Q1' AS q1, 'Q2' AS q2, 'Q3' AS q3, 'Q4' AS q4));

SELECT region, quarter, amount
FROM   sales_wide
UNPIVOT (amount FOR quarter IN (q1 AS 'Q1', q2 AS 'Q2', q3 AS 'Q3', q4 AS 'Q4'));
```

Other tools: conditional aggregation (`SUM(CASE WHEN ... END)`),
`APPROX_COUNT_DISTINCT` / `APPROX_PERCENTILE` (12c+) for large data sets,
`MODEL` clause for spreadsheet-like calculations.

## Row Limiting & Top-N

```sql
-- 12c+
SELECT * FROM orders ORDER BY order_date DESC FETCH FIRST 10 ROWS WITH TIES;
SELECT * FROM orders ORDER BY order_id OFFSET 100 ROWS FETCH NEXT 50 ROWS ONLY;

-- Pre-12c / top-N per group
SELECT * FROM (SELECT o.*, ROW_NUMBER() OVER (PARTITION BY customer_id
                                               ORDER BY order_date DESC) rn
               FROM orders o)
WHERE rn <= 3;
```

For deep pagination on large tables prefer keyset pagination
(`WHERE order_id > :last_seen_id ORDER BY order_id FETCH FIRST 50 ROWS ONLY`).
