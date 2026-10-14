# DECISIONS.md

1. **Indexes.** On `employees`, I created unique `emp_code` for identity, `(department, emp_code)` for filtered sorted listings, and `(department, joined_on)` for headcount lookups. On `attendance_logs`, I created unique `(emp_code, date)` for natural key uniqueness and monthly queries, `(date, emp_code)` and `(status, date, emp_code)` for sorted pagination and scans, and `(emp_code, punch_in)` for punch-out resolution. I rejected a standalone index on `status` because its low cardinality offers poor selectivity; the compound index covers filtering and sorting together.

2. **Punch-in race.** Both requests validate their payload and execute an atomic `insert_one`. Because `(emp_code, date)` is guarded by a unique compound index, MongoDB's storage engine accepts exactly one document. The other request triggers a `DuplicateKeyError`, which my code intercepts and maps cleanly to HTTP 409 Conflict without check-then-insert races.

3. **Ties.** All tied employees are returned, even if this exceeds `limit`. The pipeline uses `$setWindowFields` with `$rank` to compute dense competition ranks (e.g. 1, 2, 2, 4), and then filters by `rank <= limit`. The slice cutoff applies to rank value, not row count.

4. **Headcount.** The aggregation pipeline starts from `employees` rather than logs. It filters active staff by `joined_on <= month_end`, and then uses a left `$lookup` into `attendance_logs`. Employees with zero records retain an empty array and are counted in `headcount`.

5. **100x scaling.** At 100x volume (10M+ records), running real-time aggregations over raw punch records would saturate CPU and I/O. I would introduce pre-aggregated monthly summary collections updated atomically during punch-in, punch-out, and regularization, converting analytical reads from heavy multi-stage pipelines into instantaneous single-document key-value lookups.
