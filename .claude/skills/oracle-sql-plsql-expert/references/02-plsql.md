# Advanced PL/SQL Programming

## Procedures & Functions

```sql
CREATE OR REPLACE FUNCTION get_order_total (
    p_order_id IN orders.order_id%TYPE
) RETURN NUMBER
    RESULT_CACHE
IS
    l_total NUMBER;
BEGIN
    SELECT SUM(quantity * unit_price)
    INTO   l_total
    FROM   order_items
    WHERE  order_id = p_order_id;

    RETURN NVL(l_total, 0);
END get_order_total;
/
```

- Anchor types with `%TYPE` / `%ROWTYPE` so code survives column changes.
- Parameter modes: `IN` (by reference, read-only), `OUT`/`IN OUT` (by value —
  add `NOCOPY` for large collections, accepting that partial changes may be
  visible after an exception).
- Functions called from SQL: avoid side effects; mark `DETERMINISTIC` only if
  truly deterministic; use `PRAGMA UDF` (12c+) or `WITH FUNCTION` to reduce
  SQL↔PL/SQL context switches; scalar subquery caching
  (`(SELECT fn(x) FROM dual)`) helps when inputs repeat.
- `RESULT_CACHE` for read-mostly lookups; `AUTHID CURRENT_USER` vs.
  `DEFINER` for invoker's/definer's rights.

## Packages

```sql
CREATE OR REPLACE PACKAGE order_api AUTHID DEFINER IS
    -- Public types and constants
    SUBTYPE t_status IS VARCHAR2(20);
    c_status_new    CONSTANT t_status := 'NEW';
    c_status_closed CONSTANT t_status := 'CLOSED';

    e_invalid_order EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_invalid_order, -20001);

    PROCEDURE create_order (p_customer_id IN  NUMBER,
                            p_order_id    OUT NUMBER);
    PROCEDURE close_order  (p_order_id    IN  NUMBER);
    FUNCTION  order_status (p_order_id    IN  NUMBER) RETURN t_status;
END order_api;
/

CREATE OR REPLACE PACKAGE BODY order_api IS
    -- Private helper: not visible outside the body
    PROCEDURE assert_exists (p_order_id IN NUMBER) IS
        l_dummy NUMBER;
    BEGIN
        SELECT 1 INTO l_dummy FROM orders WHERE order_id = p_order_id;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20001, 'Order ' || p_order_id || ' not found');
    END assert_exists;

    PROCEDURE create_order (p_customer_id IN NUMBER, p_order_id OUT NUMBER) IS
    BEGIN
        INSERT INTO orders (customer_id, status, created_at)
        VALUES (p_customer_id, c_status_new, SYSTIMESTAMP)
        RETURNING order_id INTO p_order_id;
    END create_order;

    PROCEDURE close_order (p_order_id IN NUMBER) IS
    BEGIN
        assert_exists(p_order_id);
        UPDATE orders SET status = c_status_closed WHERE order_id = p_order_id;
    END close_order;

    FUNCTION order_status (p_order_id IN NUMBER) RETURN t_status IS
        l_status t_status;
    BEGIN
        SELECT status INTO l_status FROM orders WHERE order_id = p_order_id;
        RETURN l_status;
    END order_status;
END order_api;
/
```

Design guidance:

- **Spec = contract, body = implementation.** Changing only the body does not
  invalidate dependents.
- Group by business capability (`order_api`, `billing_api`), not by object type.
- Layering: *API packages* (business rules, transaction control) → *data access
  packages* (table APIs) → *utility packages* (logging, errors, constants).
- Package state is per session; be careful with connection pools
  (ORA-04068 "existing state of packages has been discarded" after redeploy).
  Use `PRAGMA SERIALLY_REUSABLE` or keep packages stateless where possible.
- Overloading, forward declarations, and initialization sections are available
  but keep them simple.

## Triggers

```sql
CREATE OR REPLACE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
BEGIN
    INSERT INTO orders_audit (order_id, action, old_status, new_status, changed_by, changed_at)
    VALUES (COALESCE(:NEW.order_id, :OLD.order_id),
            CASE WHEN INSERTING THEN 'I' WHEN UPDATING THEN 'U' ELSE 'D' END,
            :OLD.status, :NEW.status,
            SYS_CONTEXT('USERENV', 'SESSION_USER'), SYSTIMESTAMP);
END;
/
```

- Types: row/statement level, `BEFORE`/`AFTER`, `INSTEAD OF` (views),
  `COMPOUND` (11g+), DDL and database event triggers (`AFTER LOGON`, `SERVERERROR`).
- **Mutating table (ORA-04091)**: a row trigger cannot query its own table.
  Fix with a compound trigger that collects keys in the row section and
  processes them in `AFTER STATEMENT`.
- Keep triggers thin: auditing, derived columns, and integrity that constraints
  cannot express. Avoid business logic and cascading trigger chains — they hide
  behavior and hurt bulk performance.

```sql
CREATE OR REPLACE TRIGGER trg_order_items_total
    FOR INSERT OR UPDATE OR DELETE ON order_items
    COMPOUND TRIGGER

    TYPE t_ids IS TABLE OF orders.order_id%TYPE INDEX BY PLS_INTEGER;
    g_ids t_ids;

    AFTER EACH ROW IS
    BEGIN
        g_ids(g_ids.COUNT + 1) := COALESCE(:NEW.order_id, :OLD.order_id);
    END AFTER EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        FORALL i IN 1 .. g_ids.COUNT
            UPDATE orders o
            SET    o.total = (SELECT NVL(SUM(quantity * unit_price), 0)
                              FROM   order_items
                              WHERE  order_id = o.order_id)
            WHERE  o.order_id = g_ids(i);
    END AFTER STATEMENT;
END trg_order_items_total;
/
```

## Explicit & Implicit Cursors

```sql
-- Implicit cursor: single-row SELECT INTO, DML attributes
UPDATE employees SET salary = salary * 1.05 WHERE department_id = 10;
DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' rows updated');

-- Cursor FOR loop: auto open/fetch/close (auto bulk-fetches 100 rows at a time in 10g+)
FOR r IN (SELECT employee_id, salary FROM employees WHERE department_id = 10) LOOP
    NULL;
END LOOP;

-- Explicit parameterized cursor
DECLARE
    CURSOR cur_emp (p_dept NUMBER) IS
        SELECT employee_id, salary FROM employees WHERE department_id = p_dept;
    l_rec cur_emp%ROWTYPE;
BEGIN
    OPEN cur_emp(10);
    LOOP
        FETCH cur_emp INTO l_rec;
        EXIT WHEN cur_emp%NOTFOUND;
        -- process l_rec
    END LOOP;
    CLOSE cur_emp;
END;
/
```

- Attributes: `%FOUND`, `%NOTFOUND`, `%ROWCOUNT`, `%ISOPEN`.
- `SELECT INTO` raises `NO_DATA_FOUND` / `TOO_MANY_ROWS` — handle both.
- `SELECT ... FOR UPDATE [NOWAIT | WAIT n | SKIP LOCKED]` with
  `WHERE CURRENT OF cur` for locked row processing.
- Always close explicit cursors in exception handlers to avoid ORA-01000
  (maximum open cursors exceeded).

## REF CURSOR

```sql
CREATE OR REPLACE FUNCTION get_orders (
    p_customer_id IN NUMBER,
    p_status      IN VARCHAR2 DEFAULT NULL
) RETURN SYS_REFCURSOR
IS
    l_rc SYS_REFCURSOR;
BEGIN
    OPEN l_rc FOR
        SELECT order_id, status, created_at
        FROM   orders
        WHERE  customer_id = p_customer_id
        AND    (p_status IS NULL OR status = p_status)
        ORDER  BY created_at DESC;
    RETURN l_rc;
END get_orders;
/
```

- Strong (`TYPE t_rc IS REF CURSOR RETURN orders%ROWTYPE`) vs. weak
  (`SYS_REFCURSOR`).
- Main use: returning result sets to Java/.NET/Python clients; the client must
  close it.
- Dynamic: `OPEN l_rc FOR l_sql USING p_bind1, p_bind2;`
- 12c+ implicit result sets: `DBMS_SQL.RETURN_RESULT(l_rc);`

## Collections & Records

| Type | Indexed by | Bounded | Sparse | SQL usable | Typical use |
|------|-----------|---------|--------|------------|-------------|
| Associative array | `PLS_INTEGER` / `VARCHAR2` | No | Yes | No (12c+: in PL/SQL `TABLE()` from package types) | In-memory lookups, caches |
| Nested table | Integer | No | Can become | Yes (schema type) | BULK COLLECT, `TABLE()` operator, `MULTISET` ops |
| VARRAY | Integer | Yes | No | Yes | Small, ordered, fixed max |

```sql
DECLARE
    TYPE t_emp_rec IS RECORD (
        id     employees.employee_id%TYPE,
        name   employees.last_name%TYPE,
        salary employees.salary%TYPE
    );
    TYPE t_emp_tab   IS TABLE OF t_emp_rec;
    TYPE t_rate_map  IS TABLE OF NUMBER INDEX BY VARCHAR2(3);

    l_emps  t_emp_tab;
    l_rates t_rate_map;
    l_key   VARCHAR2(3);
BEGIN
    l_rates('USD') := 1;
    l_rates('EUR') := 1.08;

    l_key := l_rates.FIRST;
    WHILE l_key IS NOT NULL LOOP
        DBMS_OUTPUT.PUT_LINE(l_key || '=' || l_rates(l_key));
        l_key := l_rates.NEXT(l_key);
    END LOOP;
END;
/
```

- Methods: `COUNT`, `FIRST`, `LAST`, `NEXT`, `PRIOR`, `EXISTS`, `EXTEND`,
  `TRIM`, `DELETE`.
- Schema-level `CREATE TYPE t_num_tab AS TABLE OF NUMBER;` lets you pass lists
  into SQL: `WHERE id IN (SELECT column_value FROM TABLE(:ids))`.
- `MULTISET UNION / INTERSECT / EXCEPT`, `MEMBER OF`, `SUBMULTISET`, `CARDINALITY`.
- Pipelined table functions (`PIPE ROW`) stream rows into SQL for ETL.

## BULK COLLECT

```sql
DECLARE
    CURSOR cur_src IS
        SELECT order_id, amount FROM stg_orders WHERE processed = 'N';

    TYPE t_ids     IS TABLE OF stg_orders.order_id%TYPE;
    TYPE t_amounts IS TABLE OF stg_orders.amount%TYPE;

    l_ids     t_ids;
    l_amounts t_amounts;
    c_limit   CONSTANT PLS_INTEGER := 1000;
BEGIN
    OPEN cur_src;
    LOOP
        FETCH cur_src BULK COLLECT INTO l_ids, l_amounts LIMIT c_limit;
        EXIT WHEN l_ids.COUNT = 0;           -- NOT cur_src%NOTFOUND (would skip the last partial batch)

        FORALL i IN 1 .. l_ids.COUNT
            INSERT INTO orders_fact (order_id, amount) VALUES (l_ids(i), l_amounts(i));

        FORALL i IN 1 .. l_ids.COUNT
            UPDATE stg_orders SET processed = 'Y' WHERE order_id = l_ids(i);

        COMMIT;   -- batch commit only if restartable; committing while the cursor is open
                  -- on a table being updated risks ORA-01555 (fetch across commit)
    END LOOP;
    CLOSE cur_src;
END;
/
```

- Always use `LIMIT` for unbounded result sets to protect PGA memory
  (100–1000 is a typical sweet spot; measure).
- `BULK COLLECT` never raises `NO_DATA_FOUND` — check `.COUNT`.
- Also works with `SELECT ... BULK COLLECT INTO`, `RETURNING ... BULK COLLECT INTO`,
  and `EXECUTE IMMEDIATE ... BULK COLLECT INTO`.

## FORALL

```sql
DECLARE
    TYPE t_ids IS TABLE OF NUMBER;
    l_ids      t_ids := t_ids(101, 102, 103, 104);
    e_bulk_errors EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_bulk_errors, -24381);
BEGIN
    FORALL i IN 1 .. l_ids.COUNT SAVE EXCEPTIONS
        UPDATE accounts SET balance = balance - 100 WHERE account_id = l_ids(i);

    FOR i IN 1 .. l_ids.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(l_ids(i) || ': ' || SQL%BULK_ROWCOUNT(i) || ' rows');
    END LOOP;
EXCEPTION
    WHEN e_bulk_errors THEN
        FOR j IN 1 .. SQL%BULK_EXCEPTIONS.COUNT LOOP
            log_error(
                p_key   => l_ids(SQL%BULK_EXCEPTIONS(j).ERROR_INDEX),
                p_error => SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE));
        END LOOP;
END;
/
```

- One context switch per batch instead of per row — typically 10–50x faster
  than a row-by-row loop.
- Sparse collections: `FORALL i IN INDICES OF l_tab` or `VALUES OF l_idx_tab`.
- A single DML statement per `FORALL`.
- For very large volumes, compare against pure SQL (`INSERT /*+ APPEND */ ... SELECT`),
  parallel DML, or `DBMS_PARALLEL_EXECUTE` chunking by rowid.

## Dynamic SQL

```sql
-- Native dynamic SQL with binds
EXECUTE IMMEDIATE
    'UPDATE ' || DBMS_ASSERT.SQL_OBJECT_NAME(p_table) ||
    ' SET status = :1 WHERE id = :2'
    USING p_status, p_id;

EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM orders WHERE customer_id = :c'
    INTO l_count USING p_customer_id;

-- DDL
EXECUTE IMMEDIATE 'TRUNCATE TABLE stg_orders';
```

- **Always bind values**; only object names are concatenated — and those are
  validated with `DBMS_ASSERT` (`SIMPLE_SQL_NAME`, `SQL_OBJECT_NAME`,
  `ENQUOTE_NAME`) to prevent SQL injection.
- `DBMS_SQL` when the number/types of binds or columns are unknown at compile
  time (method 4); convert with `DBMS_SQL.TO_REFCURSOR` / `TO_CURSOR_NUMBER`.
- Dynamic SQL defeats compile-time dependency checks — keep it isolated and
  well tested.

## Exception Handling

```sql
CREATE OR REPLACE PROCEDURE transfer_funds (
    p_from IN NUMBER, p_to IN NUMBER, p_amount IN NUMBER
) IS
    e_insufficient_funds EXCEPTION;
    l_balance accounts.balance%TYPE;
BEGIN
    SELECT balance INTO l_balance
    FROM   accounts WHERE account_id = p_from
    FOR UPDATE;

    IF l_balance < p_amount THEN
        RAISE e_insufficient_funds;
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20010, 'Account ' || p_from || ' not found');
    WHEN e_insufficient_funds THEN
        RAISE_APPLICATION_ERROR(-20011, 'Insufficient funds in account ' || p_from);
    WHEN OTHERS THEN
        error_pkg.log_error(p_context => 'transfer_funds',
                            p_params  => p_from || ',' || p_to || ',' || p_amount);
        RAISE;    -- never swallow
END transfer_funds;
/
```

- Predefined (`NO_DATA_FOUND`, `TOO_MANY_ROWS`, `DUP_VAL_ON_INDEX`,
  `VALUE_ERROR`, `ZERO_DIVIDE`), named via `PRAGMA EXCEPTION_INIT`, and
  user-defined via `RAISE_APPLICATION_ERROR(-20000..-20999, msg, TRUE)`.
- Capture the full stack: `DBMS_UTILITY.FORMAT_ERROR_STACK`,
  `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` (line numbers), or `UTL_CALL_STACK` (12c+).
- `WHEN OTHERS THEN NULL` is a defect (compiler warning PLW-06009). Log and re-raise.
- Statement-level rollback: an unhandled exception rolls back the failing
  statement, not the transaction — the caller decides commit/rollback.
- Exceptions in the declaration section are not caught by the same block's handler.

## Autonomous Transactions

```sql
CREATE OR REPLACE PROCEDURE log_error (
    p_context IN VARCHAR2,
    p_params  IN VARCHAR2 DEFAULT NULL
) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO error_log (log_id, context, params, error_stack, backtrace, logged_by, logged_at)
    VALUES (error_log_seq.NEXTVAL, p_context, p_params,
            DBMS_UTILITY.FORMAT_ERROR_STACK,
            DBMS_UTILITY.FORMAT_ERROR_BACKTRACE,
            SYS_CONTEXT('USERENV', 'SESSION_USER'), SYSTIMESTAMP);
    COMMIT;   -- required, else ORA-06519
END log_error;
/
```

- Legitimate uses: error/audit logging that must survive the caller's rollback,
  sequence-like counters, and calling DDL from a trigger.
- Misuse: hiding mutating-table errors or committing partial business work —
  it breaks atomicity and can self-deadlock if it touches rows the parent locked.

## PL/SQL Architecture & Quality

- Compile with warnings: `ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL';`
- Optimization: `PLSQL_OPTIMIZE_LEVEL = 2` (default) or 3; native compilation
  (`PLSQL_CODE_TYPE = NATIVE`) for compute-heavy code.
- Conditional compilation (`$IF DBMS_DB_VERSION.VER_LE_12 $THEN`) for
  version-specific code and debug instrumentation.
- Instrument long-running code: `DBMS_APPLICATION_INFO.SET_MODULE/SET_ACTION`,
  `SET_SESSION_LONGOPS`, and a logging package with levels.
- Profile with `DBMS_HPROF` (hierarchical profiler) before optimizing PL/SQL.
- Edition-Based Redefinition (EBR) for zero-downtime PL/SQL deployments.
