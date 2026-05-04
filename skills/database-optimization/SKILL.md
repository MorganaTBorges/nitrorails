---
name: database-optimization
description: Use when optimizing at the PostgreSQL/MySQL layer — indexes (composite, partial, covering), EXPLAIN ANALYZE interpretation, connection pooling, VACUUM, partitioning. Distinct from activerecord-performance which operates at the ORM layer.
---

## Index Strategy

### When to add an index

Add an index when:
- A column appears in a `WHERE`, `JOIN ON`, or `ORDER BY` clause on a large table
- A foreign key column has no index (Rails does NOT add these automatically before Rails 7.2)
- A query is using `Seq Scan` on a table with more than ~10,000 rows

Do NOT add indexes blindly. Each index slows down `INSERT`, `UPDATE`, and `DELETE`. Measure first.

### Index types

**Standard B-tree index:**
```ruby
add_index :posts, :author_id
add_index :posts, :published_at
```

**Composite index — column order matters:**
```ruby
# Useful for: WHERE status = 'published' AND created_at > X
# The leading column (status) must appear in the WHERE clause
add_index :posts, [:status, :created_at]
```

Rule: put the most selective column first, OR the column that appears most often in WHERE clauses.

**Partial index — indexes only a subset of rows:**
```ruby
# Only indexes published posts — much smaller, faster for published queries
add_index :posts, :created_at, where: "status = 'published'"
add_index :users, :email, where: "deleted_at IS NULL", unique: true
```

**Covering index — includes extra columns to avoid table lookups:**
```ruby
# Query: SELECT id, title FROM posts WHERE author_id = X ORDER BY created_at
# This index covers the query entirely — no table row lookup needed
add_index :posts, [:author_id, :created_at], include: [:id, :title]
# Note: `include:` option requires PostgreSQL 11+
```

### Reading EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM posts WHERE author_id = 42 ORDER BY created_at DESC LIMIT 10;
```

Key nodes to identify:

| Node | Meaning |
|------|---------|
| `Seq Scan` | Full table scan — check if an index would help |
| `Index Scan` | Using an index — good |
| `Index Only Scan` | Covering index hit — best |
| `Bitmap Heap Scan` | Used for IN queries or OR conditions |
| `Nested Loop` | Good for small result sets |
| `Hash Join` | Good for large joins |
| `Sort` | Sorting in memory or on disk — check for index on sort column |

Red flags:
- `rows=50000` when `actual rows=1` — stale statistics, run `ANALYZE table_name`
- `Execution Time` >> `Planning Time` — query is doing real work, not just planning
- `Buffers: shared hit=0, read=10000` — no buffer cache hits, cold data

### Fixing stale statistics

```sql
-- Update statistics for the query planner
ANALYZE posts;
ANALYZE VERBOSE posts;  -- with output

-- Auto-analyze threshold: triggered when > 20% + 50 rows changed
-- Can be tuned per table:
ALTER TABLE posts SET (autovacuum_analyze_scale_factor = 0.05);
```

## Connection Pooling

### Application-level pooling (database.yml)

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  checkout_timeout: 5
```

Rule: `pool` size >= number of threads per process (Puma `RAILS_MAX_THREADS`).

If using multiple dynos/workers: each process has its own pool. Total DB connections = processes x pool size.

### PgBouncer (connection pooling proxy)

Add PgBouncer when total connections exceed PostgreSQL's `max_connections` (default: 100).

PgBouncer modes:
- **Transaction mode** — recommended for Rails. Connection returned to pool after each transaction.
- **Session mode** — connection held for entire session. Use if transaction mode causes issues with `SET`, advisory locks, or `LISTEN/NOTIFY`.

With PgBouncer in transaction mode, disable `prepared_statements` in Rails:
```yaml
# config/database.yml
production:
  prepared_statements: false
  advisory_locks: false
```

## VACUUM and Bloat

PostgreSQL's MVCC leaves dead tuples after updates and deletes. Too many dead tuples slow queries.

```sql
-- Check bloat on a table
SELECT
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE relname = 'your_table';
```

If `dead_pct` > 10%:
```sql
VACUUM ANALYZE your_table;
VACUUM VERBOSE ANALYZE your_table;  -- with output
```

For heavily updated tables, tune autovacuum:
```sql
ALTER TABLE high_churn_table SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_analyze_scale_factor = 0.01
);
```

## Partitioning

Partition large tables to improve query performance by limiting the rows scanned.

```ruby
# Rails migration (PostgreSQL range partitioning)
create_table :events, primary_key: [:id, :created_at], force: :cascade do |t|
  t.bigint :id, null: false
  t.string :type
  t.jsonb :payload
  t.timestamps null: false
end

execute <<-SQL
  ALTER TABLE events PARTITION BY RANGE (created_at);
  CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
  CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
SQL
```

Use partitioning when:
- Table has > 10M rows and queries filter on a date/range column
- You need to drop old data quickly (drop partition instead of DELETE)
