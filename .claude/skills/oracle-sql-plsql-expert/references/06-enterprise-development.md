# Enterprise Development

## Code Review Checklist (Oracle SQL & PL/SQL)

**Correctness**
- [ ] Join conditions complete (no accidental cartesian products)
- [ ] NULL handling: `NOT IN` with nullable columns, `=` comparisons with NULL, `NVL` defaults
- [ ] `SELECT INTO` handles `NO_DATA_FOUND` / `TOO_MANY_ROWS`
- [ ] Outer-join predicates in the right clause
- [ ] Date logic: time components, time zones, `TRUNC` usage
- [ ] Transaction boundaries clear; no commits inside reusable units

**Performance**
- [ ] Set-based where possible; no row-by-row DML loops ("slow-by-slow")
- [ ] BULK COLLECT has `LIMIT`; FORALL used for batch DML
- [ ] Bind variables in dynamic SQL; no literal concatenation of values
- [ ] Predicates are sargable (no functions/implicit conversions on indexed columns)
- [ ] New queries checked with an execution plan on realistic volumes
- [ ] Foreign keys indexed; new indexes justified

**Robustness & security**
- [ ] No `WHEN OTHERS THEN NULL`; errors logged with backtrace and re-raised
- [ ] Explicit cursors closed in exception paths
- [ ] Dynamic identifiers validated with `DBMS_ASSERT`
- [ ] Least-privilege grants; no hard-coded credentials
- [ ] Re-runnable deployment scripts with rollback

**Maintainability**
- [ ] Naming and formatting standards followed
- [ ] `%TYPE` / `%ROWTYPE` anchoring; constants instead of magic values
- [ ] Logic in packages with a clear public API
- [ ] Unit tests added/updated
- [ ] Comments explain *why*, not *what*

Give review feedback that teaches: point to the risk, show the better pattern,
and distinguish blocking issues from suggestions.

## Coding Standards

| Element | Convention | Example |
|---------|-----------|---------|
| Keywords | UPPERCASE | `SELECT`, `BEGIN` |
| Identifiers | lowercase snake_case | `order_items` |
| Parameters | `p_` | `p_order_id` |
| Local variables | `l_` | `l_total` |
| Package globals | `g_` | `g_debug_enabled` |
| Constants | `c_` | `c_max_retries` |
| Types | `t_` | `t_order_tab` |
| Cursors | `cur_` | `cur_pending_orders` |
| Exceptions | `e_` | `e_invalid_status` |
| Primary key | `pk_<table>` | `pk_orders` |
| Foreign key | `fk_<table>_<ref>` | `fk_orders_cust` |
| Unique / Check | `uk_` / `ck_` | `uk_orders_no` |
| Indexes | `ix_<table>_<cols>` | `ix_orders_customer` |
| Sequences | `<table>_seq` | `orders_seq` |
| Triggers | `trg_<table>_<purpose>` | `trg_orders_audit` |

Rules: one statement per line, explicit column lists in `INSERT`, no
`SELECT *` in production code, named notation for calls with many parameters
(`p_order_id => l_id`), end labels on program units (`END order_api;`),
compile with `PLSQL_WARNINGS` enabled and treat warnings seriously.

## Unit Testing (utPLSQL)

```sql
CREATE OR REPLACE PACKAGE test_order_api IS
    --%suite(Order API)
    --%rollback(auto)

    --%test(Creates an order with status NEW)
    PROCEDURE create_order_sets_new_status;

    --%test(Closing a missing order raises -20001)
    --%throws(-20001)
    PROCEDURE close_missing_order_raises;
END test_order_api;
/

CREATE OR REPLACE PACKAGE BODY test_order_api IS
    PROCEDURE create_order_sets_new_status IS
        l_order_id NUMBER;
    BEGIN
        order_api.create_order(p_customer_id => 1, p_order_id => l_order_id);
        ut.expect(order_api.order_status(l_order_id)).to_equal(order_api.c_status_new);
    END create_order_sets_new_status;

    PROCEDURE close_missing_order_raises IS
    BEGIN
        order_api.close_order(p_order_id => -1);
    END close_missing_order_raises;
END test_order_api;
/

EXEC ut.run('test_order_api');
```

- Test happy path, boundaries, NULLs, error paths, and concurrency-sensitive logic.
- Compare result sets: `ut.expect(l_actual_cursor).to_equal(l_expected_cursor);`
- Run in CI (utPLSQL-cli) with code coverage reports; tests auto-rollback.
- Performance regression tests: assert on elapsed time or logical reads for
  critical SQL with representative data.

## Technical Documentation

- **Package header comments**: purpose, owner, dependencies, change history
  (or rely on version control), and usage examples.
- **Design docs**: data model diagram, data flow, batch schedules and
  dependencies, error-handling and restart strategy, SLAs.
- **Runbooks**: how to monitor, restart failed jobs, common errors and their
  fixes, escalation contacts.
- **Release notes**: objects changed, scripts and order, rollback steps,
  verification queries.
- Keep the data dictionary meaningful: `COMMENT ON TABLE/COLUMN`.

## Application–Database Integration

- Expose **PL/SQL APIs** (packages) instead of direct table DML — enforces
  business rules and allows internal change without app redeploys ("thick
  database" / SmartDB approach).
- Return result sets via `SYS_REFCURSOR`; accept lists via collection types or
  JSON (`JSON_TABLE`, `JSON_OBJECT`, `JSON_ARRAYAGG`).
- Connection pools: set `DBMS_APPLICATION_INFO` module/action and
  `CLIENT_IDENTIFIER` for traceability; avoid relying on package state; use
  statement caching on the client.
- Batch at the client (JDBC `addBatch`, array binding) to reduce round trips;
  set fetch size appropriately.
- Integration mechanisms: database links, REST via ORDS, Advanced Queuing /
  Transactional Event Queues, `DBMS_SCHEDULER` jobs, external tables, `UTL_HTTP`.
- Error contract: define application error codes (-20000..-20999) and messages
  the app layer can map to user-facing responses.

## Collaboration with Application Developers

- Review ORM-generated SQL (N+1 queries, missing binds, `SELECT *`, unbounded fetches).
- Share execution plans and explain trade-offs in plain language.
- Agree on API contracts, error codes, and transaction ownership early.
- Provide test data sets and performance baselines for new features.

## SDLC

Requirements → design (data model, API spec) → development (standards,
version control, feature branches) → code review → unit/integration testing →
performance testing on production-like volumes → UAT → release
(change management, rollback plan) → hypercare → production support.
Works in Agile/Scrum (story estimation, sprint demos) and in waterfall
regulated environments alike.

## Production Support

- Monitor batch jobs (`DBA_SCHEDULER_JOB_RUN_DETAILS`), error logs, and
  key business metrics; alert on failures and SLA breaches.
- Classify incidents by severity; follow runbooks; communicate status
  regularly to stakeholders.
- Track recurring issues as problems and eliminate them through permanent fixes.
- Hand over with clear notes: what was observed, what was done, what remains.

## Mentoring Junior Developers

- Teach the progression: correct SQL → set-based thinking → reading plans →
  bulk processing → design.
- Use code reviews as teaching moments with concrete before/after examples.
- Pair on production incidents so juniors learn the diagnostic method, not just the fix.
- Maintain a shared knowledge base: standards, templates (package skeleton,
  deployment script, test package), and common pitfalls.
- Encourage measuring before optimizing and asking "what does the plan say?"
