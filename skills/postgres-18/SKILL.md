---
name: postgres-18
description: "PostgreSQL 18/19 facts that pre-2026-cutoff models get wrong. Use when writing Postgres SQL, DDL, migrations, EXPLAIN analysis, or Postgres Docker/Compose config. Corrects uuidv7(), async I/O (io_method), B-tree skip scan, virtual generated columns, RETURNING OLD/NEW, temporal constraints, NOT NULL NOT VALID, and the changed Docker data directory."
---

# PostgreSQL 18+ (GA 2025-09)

Current: 18.x line (18.7, 2026-11-12). PostgreSQL 19 GA was planned 2026-11-19 — check the release notes before asserting or denying 19 features.

## Stale → current

| Stale habit (pre-2026 models) | Current (18+) |
| --- | --- |
| "Postgres has no UUIDv7 — use `gen_random_uuid()`, pgcrypto, or an extension" | Built-in `uuidv7()`; `uuidv4()` is now an alias for `gen_random_uuid()` |
| `GENERATED ALWAYS AS (...)` is stored | **VIRTUAL is now the default**; write `STORED` explicitly for old behavior |
| `EXPLAIN (ANALYZE, BUFFERS)` to see buffers | `EXPLAIN ANALYZE` includes buffers by default |
| "Composite index is useless without a leading-column predicate" | B-tree **skip scan**, planner-automatic |
| Extra SELECT around a mutation to see old+new values | `RETURNING old.x, new.x` |
| App-side overlap checks for bookings/validity windows | Temporal PK/UNIQUE/FK (`WITHOUT OVERLAPS`, `PERIOD`) |
| Adding `NOT NULL` = full-table scan + strong lock | `SET NOT NULL NOT VALID` + `VALIDATE CONSTRAINT` later |
| Compose volume at `/var/lib/postgresql/data` | Data dir **moved** — see Docker gotcha below |
| `effective_io_concurrency` default 1, meaningless without fadvise | Default **16**, meaningful everywhere |

## uuidv7

```sql
CREATE TABLE items (
  id uuid PRIMARY KEY DEFAULT uuidv7()
);
SELECT uuid_extract_timestamp(id) FROM items;  -- works for v7 (and v1); NULL otherwise
SELECT uuidv7('5 years'::interval);            -- backdated, still time-ordered keys
```
Time-ordered keys keep B-tree PK indexes compact (right-edge inserts) — no random-v4 fragmentation.

## Async I/O

```sql
SHOW io_method;   -- sync | worker (default) | io_uring (needs --with-liburing build)
-- io_workers: default 3 (set at server start; REMOVED in 19 → auto-scaling io_min/io_max_workers)
SELECT * FROM pg_aios;   -- in-flight AIO handles
```
Biggest wins: seq scans, bitmap heap scans, VACUUM on network-attached disks.

## Virtual generated columns

```sql
CREATE TABLE t (a int, total numeric GENERATED ALWAYS AS (a * 2));        -- VIRTUAL (new default)
CREATE TABLE t (a int, total numeric GENERATED ALWAYS AS (a * 2) STORED); -- PG 12–17 behavior
```
Gotchas in 18: **no indexes on virtual columns** (support lands in 19); no in-place virtual↔stored conversion (drop + re-add); built-in functions only in the expression; no statistics on them.

## RETURNING OLD/NEW

```sql
UPDATE products SET price = price * 1.10
RETURNING name, old.price AS old_price, new.price AS new_price;
```
`old` is NULL for plain INSERT, `new` is NULL for DELETE; both non-NULL in `ON CONFLICT DO UPDATE`.

## Temporal constraints

```sql
CREATE TABLE emp (
  emp_id   int,
  valid_at daterange,
  PRIMARY KEY (emp_id, valid_at WITHOUT OVERLAPS)   -- last column; range type; enforced as GiST
);
CREATE TABLE job (
  emp_id   int,
  valid_at daterange,
  FOREIGN KEY (emp_id, PERIOD valid_at)
    REFERENCES emp (emp_id, PERIOD valid_at)
);
```
Requires `btree_gist` for scalar key columns; temporal FK requires the referenced table to have a `WITHOUT OVERLAPS` key; no CASCADE/SET NULL referential actions.

## NOT NULL NOT VALID (zero-downtime adds)

```sql
ALTER TABLE t ALTER COLUMN c SET NOT NULL NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT <name>;   -- later; SHARE UPDATE EXCLUSIVE lock
```

## Docker / Compose gotcha

PG 18 moved the data dir: the cluster lives at `/var/lib/postgresql/18/docker` and the image's `VOLUME` is now **`/var/lib/postgresql`** (was `/var/lib/postgresql/data`).

```yaml
services:
  postgres:
    image: postgres:18-alpine
    volumes:
      - pgdata:/var/lib/postgresql    # NOT /var/lib/postgresql/data
volumes:
  pgdata:
```
A legacy volume mounted at `/var/lib/postgresql/data` is silently ignored — you get a fresh empty DB.

## Other 18 behavior changes

- `initdb` enables **data checksums by default** (`--no-data-checksums` to opt out); pg_upgrade requires matching checksum settings on both ends — and now **retains optimizer statistics**, so no post-upgrade `ANALYZE`.
- MD5 password auth deprecated (warns on `CREATE`/`ALTER ROLE`).
- `COPY FROM` CSV: `\.` no longer ends input unless alone on its line.
- RESTRICT FK violations report SQLSTATE **23001** (was 23503).
- Unlogged partitioned tables disallowed.

## PG 19 (GA ~2026-11-19) — do not attribute these to 18

`REPACK [CONCURRENTLY]`, `WAIT` (read-your-writes on standbys), sequence logical replication, `INSERT ... ON CONFLICT DO SELECT ... RETURNING`, JIT off by default, `default_toast_compression = lz4`, `io_workers` removed in favor of auto-scaling I/O workers.

## Sources

- https://www.postgresql.org/docs/18/release-18.html (2025-09-25)
- https://www.postgresql.org/docs/18/functions-uuid.html
- https://www.postgresql.org/docs/18/runtime-config-resource.html (io_method)
- https://www.postgresql.org/docs/18/ddl-generated-columns.html
- https://www.postgresql.org/docs/18/dml-returning.html
- https://www.postgresql.org/about/news/postgresql-187-1712-1616-1520-1425-and-19-rc-2-released-3442/ (2026-11-12)
- https://www.postgresql.org/docs/19/release-19.html
- Docker data dir change: https://github.com/docker-library/postgres/pull/1259
