---
name: systematic-diagnosis
description: Use before any code change to identify the real Rails performance bottleneck — distinguishes database, memory, CPU, I/O, and connection-layer problems. Prevents optimizing the wrong layer.
---

<HARD-GATE>
Do NOT recommend or implement any code change before completing this diagnosis.
The bottleneck layer must be identified and stated explicitly before any optimization begins.
</HARD-GATE>

## Goal

Identify which layer is responsible for the performance problem. Optimizing the wrong layer wastes time and adds complexity without improving production performance.

## Decision Tree

### Step 1: Read the baseline data

From `profiling-first`:

| Signal | Likely Layer |
|--------|-------------|
| High SQL query count (> 10 per request) | Database — N+1 |
| High SQL total time with low query count | Database — slow queries, missing indexes |
| High memory allocation or growing RSS | Memory layer |
| High response time, low SQL count, normal memory | Ruby/CPU layer |
| Fast in isolation, slow under load | Connections/concurrency |
| Slow only for large datasets | Database — pagination, iteration |

### Step 2A: Database — N+1 detection

Enable Bullet in development:

```ruby
# Gemfile (development, test groups)
gem 'bullet'

# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true
  Bullet.rails_logger  = true
  Bullet.add_footer    = true
end
```

Restart the server. Reproduce the slow request. Bullet logs which associations need preloading.

Also check rack-mini-profiler SQL tab for repeating query patterns (same query executing N times).

If N+1 confirmed → invoke `activerecord-performance`.

### Step 2B: Database — slow query, missing index

Run `EXPLAIN ANALYZE` on the slow query:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM your_table WHERE your_condition;
```

Red flags:
- `Seq Scan` on a large table (should be `Index Scan`)
- `rows=10000` when `actual rows=5` (bad statistics — run `ANALYZE table_name`)
- `Execution Time` > 100ms

If slow query / missing index confirmed → invoke `database-optimization`.

### Step 2C: Memory layer

Profile memory allocation:

```ruby
require 'memory_profiler'
report = MemoryProfiler.report { [reproduce the slow operation] }
report.pretty_print(to_file: 'tmp/memory_report.txt')
```

Key questions:
- What object type is allocated most? (String, Array, Hash, ActiveRecord::Base)
- Is it application code or a gem?
- Is memory retained (not GC'd) or just allocated and collected?

If memory problem confirmed → invoke `memory-optimization`.

### Step 2D: Ruby/CPU layer

Profile with flamegraph:

```
# Add pp=flamegraph to URL: http://localhost:3000/your-endpoint?pp=flamegraph
# Requires rack-mini-profiler + flamegraph + stackprof gems
```

Wide bars that are NOT SQL calls are Ruby CPU time.
Common causes: complex view rendering, serialization, regex, sorting large arrays.

If caching could eliminate the work → invoke `caching-strategies`.
If it's heavy computation → this is a Ruby optimization problem.

### Step 2E: Connection/concurrency layer

Check database connection pool vs. worker count:

```ruby
# config/database.yml
pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

If web server uses more threads than the pool allows, requests wait for connections.
Rule: pool size ≥ web server thread count.

If concurrency problem confirmed → invoke `database-optimization` (connection pooling section).

## Output

Before moving to optimization, state explicitly:

```
Diagnosis complete:
Bottleneck layer:  [database-n+1 / database-slow-query / memory / ruby-cpu / connections]
Root cause:        [specific description]
Recommended skill: [which domain skill to invoke]
```
