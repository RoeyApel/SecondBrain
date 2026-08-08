# PostgreSQL Coding Standards

## Core Principles

- Design schemas around data integrity first; let the database enforce invariants via constraints, not application code alone.
- Treat `EXPLAIN (ANALYZE, BUFFERS)` as the source of truth for performance — never guess.
- Prefer PostgreSQL-native features (constraints, RLS, generated columns) over reimplementing them in application logic.

## Architecture & Structure

- Normalize schemas by default; denormalize only after measuring a real query bottleneck.
- Enforce data integrity with `NOT NULL`, `CHECK`, `UNIQUE`, and `FOREIGN KEY` constraints rather than relying on application-level validation alone.
- Use one clear naming convention (`snake_case` for tables/columns) and apply it consistently across the schema.
- Name foreign keys, indexes, and constraints explicitly rather than relying on auto-generated names, so migrations and error messages stay readable.
- Prefer `timestamptz` over `timestamp` for all time-related columns — `timestamp` silently discards time zone information.
- Prefer `text` (optionally with a `CHECK (length(...) <= n)` constraint) over `varchar(n)` for variable-length strings; PostgreSQL gives no storage or performance benefit to `varchar(n)` over `text`.
- Use `numeric` for exact values (money, quantities); never use `float`/`double precision` for values requiring exact arithmetic.
- Model multi-tenant or per-user data access with Row-Level Security (RLS) rather than filtering solely in application code.

## Code Quality & Style

- Write explicit column lists in `INSERT`/`UPDATE`/`SELECT`; never rely on positional or `SELECT *` behavior in application code or views.
- Keep migrations small, single-purpose, and forward-only; avoid bundling schema and large data changes in one migration.
- Use `IF EXISTS` / `IF NOT EXISTS` in DDL to keep migrations idempotent where feasible.
- Wrap multi-statement schema changes in a transaction where the DDL supports it, so a failed migration doesn't leave the schema half-changed.
- Add comments (`COMMENT ON TABLE/COLUMN`) for non-obvious business rules encoded in constraints.

## Error Handling

- Use parameterized queries / prepared statements exclusively — never build SQL via string concatenation or interpolation.
- Let constraint violations (`UNIQUE`, `FOREIGN KEY`, `CHECK`) fail loudly and be handled by the caller; do not pre-check-then-write (check-then-act races under concurrency) when a constraint can enforce it atomically.
- On large tables, avoid migrations that take an `ACCESS EXCLUSIVE` lock for long periods (e.g., adding a column with a volatile default, adding a `NOT NULL` without a prior `CHECK`); use PostgreSQL's documented patterns for online schema changes instead.
- Use explicit transaction boundaries (`BEGIN`/`COMMIT`/`ROLLBACK`) for multi-statement operations that must be atomic.

## Security

- Never construct queries by concatenating user input; always use parameterized queries to prevent SQL injection.
- Apply least-privilege role design: create separate roles per application/service (e.g., `app_reader`, `app_writer`) and grant only the privileges each needs, rather than using a superuser or table-owner role for application connections.
- Use role membership to compose privileges instead of duplicating `GRANT` statements across many roles.
- Enable Row-Level Security (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`) for tables holding per-tenant or per-user data, and define explicit policies for each command type (`SELECT`, `UPDATE`, etc.) rather than one broad policy.
- Remember RLS is bypassed by superusers, roles with `BYPASSRLS`, and (by default) table owners — never assume RLS alone secures administrative connections.
- Keep RLS policy expressions simple (direct column comparisons); avoid subqueries in policies where possible, and be aware they can introduce race conditions under `READ COMMITTED` isolation.
- Require SSL/TLS for client connections handling sensitive data, per connection security settings.
- Never grant privileges to `PUBLIC` on sensitive objects.

## Performance

- Index columns used in `WHERE`, `JOIN`, and `ORDER BY` clauses, but do not index speculatively — every index adds write overhead and must be justified by a query pattern.
- Order columns in multicolumn (composite) indexes by selectivity and the query's filter pattern; leverage left-to-right prefix matching.
- Use partial indexes for queries that always filter on a subset condition (e.g., `WHERE status = 'active'`) instead of a full index.
- Use expression indexes when queries filter on a computed/transformed value (e.g., `LOWER(email)`).
- Use covering indexes (`INCLUDE`) when index-only scans would avoid heap lookups for hot queries.
- Regularly check for and remove redundant or unused indexes via `pg_stat_user_indexes`.
- Keep planner statistics current: run `ANALYZE` after large bulk loads, and use extended statistics for correlated multi-column predicates.
- For bulk data loads, use `COPY` instead of row-by-row `INSERT`, and consider temporarily dropping indexes/foreign keys, increasing `maintenance_work_mem`, and batching within transactions.
- Diagnose slow queries with `EXPLAIN (ANALYZE, BUFFERS)`, comparing estimated vs. actual row counts to detect stale statistics.
- Avoid N+1 query patterns from application code; prefer a single query with joins/aggregation over per-row round trips.
- Choose the appropriate index type for the data: B-tree for general equality/range, GIN for full-text/array/JSONB, BRIN for large naturally-ordered (e.g., time-series) tables.

## Testing

- Test RLS policies explicitly by executing queries as each role (`SET ROLE ...`) to verify both visibility and modification restrictions.
- Include migration tests that apply and roll back schema changes against a representative dataset size, not just an empty database.
- Verify query plans for critical/hot-path queries in CI or review (e.g., assert an index scan is used, not a sequential scan) when regressions would be costly.
- Test constraint behavior (`CHECK`, `UNIQUE`, `FOREIGN KEY`) with both valid and invalid data to confirm the database rejects bad states.

## Anti-Patterns to Avoid

- Do not build SQL strings via concatenation or f-strings with user input — always parameterize.
- Do not use `SELECT *` in application queries or views.
- Do not use `timestamp` (without time zone) for event or audit timestamps.
- Do not rely on RLS as the only access control for superuser or table-owner connections.
- Do not add unbounded/volatile-default columns or `NOT NULL` constraints to large existing tables without an online-migration strategy.
- Do not create indexes speculatively without evidence from `EXPLAIN ANALYZE` or query patterns.
- Do not use `float`/`double precision` for monetary or exact-decimal values.
- Do not grant broad privileges (e.g., superuser, `ALL PRIVILEGES` on the database) to application roles.
- Do not ignore mismatches between estimated and actual row counts in `EXPLAIN ANALYZE` output — they indicate stale statistics or a modeling problem.

## Sources

- [PostgreSQL Documentation — Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [PostgreSQL Documentation — Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL Documentation — Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [PostgreSQL Documentation — Database Roles](https://www.postgresql.org/docs/current/user-manag.html)
- [PostgreSQL Documentation — EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [PostgreSQL Documentation — Character Types](https://www.postgresql.org/docs/current/datatype-character.html)
- [PostgreSQL Documentation — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)
- [PostgreSQL Documentation — Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html)
