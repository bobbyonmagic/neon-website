---
title: 'PostgreSQL 19 New Features'
page_title: "PostgreSQL 19 New Features: What's New and Why It Matters"
page_description: 'Explore PostgreSQL 19 new features including ON CONFLICT DO SELECT, temporal data operations, pg_plan_advice query hints, the REPACK command, JSON COPY TO, and logical replication improvements.'
ogImage: ''
updatedOn: '2026-04-14T00:00:00+00:00'
enableTableOfContents: true
nextLink:
  title: 'PostgreSQL 19 ON CONFLICT DO SELECT'
  slug: 'postgresql-19/on-conflict-do-select'
---

**Summary**: PostgreSQL 19 introduces significant features including atomic get-or-create with `ON CONFLICT DO SELECT`, temporal data operations with `FOR PORTION OF`, query plan hints via `pg_plan_advice`, the `REPACK` command for online table maintenance, native JSON export with `COPY TO`, and major logical replication improvements. This overview covers the highlights with links to detailed guides.

## Introduction

PostgreSQL 19 is currently in development, with a final release expected in late 2026. This version builds on PostgreSQL 18's foundation with features that address long-standing developer requests: a proper get-or-create operation, temporal data manipulation, query plan control, and online table maintenance.

The release focuses on four areas:

- **DML improvements**: New conflict handling and temporal operations
- **Query planning**: Official plan hint support via a contrib module
- **Maintenance**: Online table repacking without downtime
- **Operations**: Better logical replication, JSON export, and monitoring

Let's look at what makes PostgreSQL 19 a significant release.

## DML and Query Improvements

### [ON CONFLICT DO SELECT](/postgresql/postgresql-19/on-conflict-do-select)

PostgreSQL 19 adds a third action to `INSERT ... ON CONFLICT`: `DO SELECT`. This gives you atomic get-or-create semantics - insert a row and get it back, or if it already exists, get the existing row. No dead tuples, no CTE workarounds.

```sql
INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email) DO SELECT
RETURNING *;
```

This replaces the common no-op `DO UPDATE SET col = EXCLUDED.col` workaround, which generated dead tuples on every conflict. Benchmarks show `DO SELECT` is nearly 4x faster than the no-op update approach.

### [Temporal Data Operations: FOR PORTION OF](/postgresql/postgresql-19/temporal-data-operations)

Building on PostgreSQL 18's `WITHOUT OVERLAPS` temporal constraints, PostgreSQL 19 adds `UPDATE ... FOR PORTION OF` and `DELETE ... FOR PORTION OF`. When you modify data within a specific time range, PostgreSQL automatically splits the row to preserve the untouched portions.

```sql
-- Original row: product 1 at $29.99 for all of 2025
-- Update price to $34.99 for Q3 only
UPDATE product_prices
FOR PORTION OF valid_range FROM '2025-07-01' TO '2025-10-01'
SET price = 34.99
WHERE product_id = 1;

-- Result: three rows
-- Jan-Jun: $29.99 (preserved automatically)
-- Jul-Sep: $34.99 (updated)
-- Oct-Dec: $29.99 (preserved automatically)
```

This completes PostgreSQL's SQL:2011 temporal feature set, making it suitable for booking systems, employee records, insurance policies, and any data with validity periods.

### GROUP BY ALL

A convenience feature that automatically groups by every non-aggregate expression in the SELECT list:

```sql
-- Before: manually repeat column names
SELECT department, role, count(*)
FROM employees
GROUP BY department, role;

-- PostgreSQL 19: GROUP BY ALL
SELECT department, role, count(*)
FROM employees
GROUP BY ALL;
```

This eliminates a common source of errors when adding or removing columns from SELECT lists.

### IGNORE NULLS / RESPECT NULLS for Window Functions

SQL-standard null handling for five window functions: `lead()`, `lag()`, `first_value()`, `last_value()`, and `nth_value()`:

```sql
-- Get the last non-null reading for each sensor
SELECT sensor_id,
    last_value(reading) IGNORE NULLS OVER (
        PARTITION BY sensor_id ORDER BY ts
    ) AS last_known_reading
FROM sensor_data;
```

`RESPECT NULLS` is the default and preserves existing behavior. `IGNORE NULLS` skips null rows when searching for a value, which is useful for time-series data with gaps.

## Query Planning and Performance

### [pg_plan_advice: Query Plan Hints](/postgresql/postgresql-19/pg-plan-advice)

PostgreSQL has historically avoided query plan hints. That changes with `pg_plan_advice`, a contrib module that provides plan stabilization and override capabilities.

The workflow: generate advice from a known-good plan, then feed it back to lock the plan.

```sql
-- Generate advice from the current plan
EXPLAIN (COSTS OFF, PLAN_ADVICE)
SELECT * FROM orders o JOIN customers c ON o.cust_id = c.id;
-- Output: JOIN_ORDER(o c)  HASH_JOIN(c)  SEQ_SCAN(o c)

-- Lock this plan
SET pg_plan_advice.advice = 'JOIN_ORDER(o c) HASH_JOIN(c) SEQ_SCAN(o c)';
```

Unlike Oracle or MySQL hints embedded in SQL comments, advice lives in a GUC setting and includes a feedback mechanism that tells you whether each hint was honored.

### Memoize Planner Estimates in EXPLAIN

EXPLAIN now shows estimated cache metrics on Memoize nodes:

```
->  Memoize  (cost=... rows=...)
      Cache Key: t.id
      Estimates: capacity=2 distinct keys=2 lookups=1000 hit percent=99.80%
```

This helps diagnose why the planner chose (or avoided) Memoize and whether `n_distinct` statistics need tuning.

## Table Maintenance

### [REPACK: Online Table Rebuilding](/postgresql/postgresql-19/repack-command)

`REPACK` absorbs the functionality of both `VACUUM FULL` and `CLUSTER` into a single command, with the addition of a `CONCURRENTLY` mode:

```sql
-- Reclaim space (like VACUUM FULL)
REPACK orders;

-- Reorder by index (like CLUSTER)
REPACK orders USING INDEX orders_created_at_idx;

-- Online mode: table stays accessible during repack
REPACK (CONCURRENTLY) orders USING INDEX orders_created_at_idx;
```

With `CONCURRENTLY`, the `ACCESS EXCLUSIVE` lock is only held briefly during the final file swap. The table remains readable and writable for the bulk of the operation.

### ALTER TABLE MERGE/SPLIT PARTITIONS

New DDL commands to restructure partitions without manual data movement:

```sql
-- Split a quarterly partition into months
ALTER TABLE sales SPLIT PARTITION sales_q1 INTO (
    PARTITION sales_jan FOR VALUES FROM ('2026-01-01') TO ('2026-02-01'),
    PARTITION sales_feb FOR VALUES FROM ('2026-02-01') TO ('2026-03-01'),
    PARTITION sales_mar FOR VALUES FROM ('2026-03-01') TO ('2026-04-01')
);

-- Merge partitions back together
ALTER TABLE sales MERGE PARTITIONS (sales_jan, sales_feb) INTO sales_jan_feb;
```

## Data Export

### [JSON Format for COPY TO](/postgresql/postgresql-19/json-copy-to)

Native JSON output support for `COPY TO`, producing NDJSON (one JSON object per line) by default or a JSON array with `FORCE_ARRAY`:

```sql
-- NDJSON output
COPY users TO STDOUT WITH (FORMAT JSON);
-- {"id":1,"email":"alice@example.com","name":"Alice"}
-- {"id":2,"email":"bob@example.com","name":"Bob"}

-- JSON array output
COPY users TO STDOUT WITH (FORMAT JSON, FORCE_ARRAY);
-- [
--  {"id":1,"email":"alice@example.com","name":"Alice"}
-- ,{"id":2,"email":"bob@example.com","name":"Bob"}
-- ]
```

This replaces the `row_to_json()` and `json_agg()` workarounds with streaming, memory-efficient output.

### COPY TO for Partitioned Tables

`COPY partitioned_table TO` now works directly, without wrapping in a subquery:

```sql
COPY partitioned_sales TO '/tmp/sales.csv' WITH (FORMAT csv);
```

About 7-8% faster than the `COPY (SELECT * FROM ...) TO` workaround.

## Logical Replication and Operations

### [Logical Replication Improvements](/postgresql/postgresql-19/logical-replication-improvements)

PostgreSQL 19 addresses several long-standing logical replication pain points:

**Sequence synchronization**: Subscribers can now sync sequence values from the publisher, preventing duplicate key errors after failover.

```sql
CREATE PUBLICATION my_pub FOR ALL TABLES, ALL SEQUENCES;
```

**EXCEPT TABLE**: Publications using `FOR ALL TABLES` can exclude specific tables:

```sql
CREATE PUBLICATION prod_pub FOR ALL TABLES
    EXCEPT (TABLE audit_log, temp_imports);
```

**Dynamic WAL level**: The `effective_wal_level` parameter adjusts automatically based on whether logical replication slots exist, eliminating the need to manually configure and restart for WAL level changes.

### pg_get_*_ddl() Functions

Three new functions for programmatic DDL extraction:

```sql
SELECT * FROM pg_get_database_ddl('mydb');
-- CREATE DATABASE mydb WITH TEMPLATE = template0 ENCODING = 'UTF8' ...

SELECT * FROM pg_get_role_ddl('app_user');
-- CREATE ROLE app_user WITH LOGIN PASSWORD '********' ...
```

Also available: `pg_get_tablespace_ddl()`. These provide a cleaner alternative to parsing `pg_dump` output when you need DDL for specific objects.

### pg_dumpall Non-Text Formats

`pg_dumpall` now supports custom (`-Fc`), directory (`-Fd`), and tar (`-Ft`) output formats:

```bash
pg_dumpall -Fc -f full-dump
pg_restore --globals-only full-dump  # restore only roles/tablespaces
```

### 64-bit MultiXactOffset

MultiXactOffset has been widened from 32-bit to 64-bit, eliminating the ~4 billion member wraparound limit. Previously, heavy use of row-level locking (concurrent `SELECT FOR UPDATE` across many transactions) could exhaust MultiXact member space, causing write failures that required emergency vacuuming. This risk is removed in PostgreSQL 19.

## Monitoring Improvements

### WAL Statistics

The `pg_stat_wal` view gains a `wal_fpi_bytes` column tracking bytes used by full-page images:

```sql
SELECT wal_records, wal_fpi, wal_fpi_bytes, wal_bytes FROM pg_stat_wal;
```

### Vacuum Progress

`pg_stat_progress_vacuum` adds `mode` (normal/aggressive/failsafe) and `started_by` (auto/manual/wraparound):

```sql
SELECT pid, relid::regclass, phase, mode, started_by
FROM pg_stat_progress_vacuum;
```

### Per-Process Logging

`log_min_messages` accepts per-process-type overrides:

```
log_min_messages = 'warning, autovacuum:debug1, archiver:debug5'
```

### psql Prompt Additions

Two new prompt escapes: `%i` shows primary/standby status, `%S` shows the current `search_path`:

```
\set PROMPT1 '[%i] %/%R%x%# '
-- Result: [primary] mydb=#
```

## Getting Started

PostgreSQL 19 is currently in development. To try these features, you can build from the PostgreSQL `master` branch or use the PGDG snapshot packages:

```bash
# Docker approach using PGDG snapshots
# See our PostgreSQL 19 guide for Dockerfile details
docker build -t pg19-dev .
docker run -d --name pg19 -p 5433:5432 pg19-dev
psql -h localhost -p 5433 -U postgres
```

## Looking Ahead

PostgreSQL 19 addresses several long-standing community requests. `ON CONFLICT DO SELECT` solves a problem that has existed since PostgreSQL 9.5. `FOR PORTION OF` completes the SQL:2011 temporal feature set. `pg_plan_advice` breaks new ground for PostgreSQL by providing official plan hints. And `REPACK (CONCURRENTLY)` brings online table maintenance into core, eliminating the need for external extensions.

The final release is expected in late 2026. We will update this page and the individual feature guides as the release progresses.
