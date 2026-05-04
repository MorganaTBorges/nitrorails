---
name: profiling-first
description: Use before any Rails performance optimization — establishes a measurable baseline using local tools, APM manual collection, or MCP integration. Hard gate: no diagnosis or optimization without a recorded baseline.
---

<HARD-GATE>
Do NOT proceed to diagnosis or optimization without a recorded baseline.
If the developer has not provided a baseline, complete this skill first.
</HARD-GATE>

## Goal

Record a measurable baseline before any code changes. This baseline is the "before" number that `verification-after-optimization` will compare against.

## Step 1: Choose the measurement layer

Work from highest to lowest availability:

### Layer 3 — MCP integration (if configured)

If New Relic MCP tools are available:
```
mcp__newrelic__execute_nrql_query:
  query: "SELECT percentile(duration, 95) FROM Transaction WHERE name = '<endpoint>' SINCE 7 days ago"
```

If Datadog MCP tools are available, query the P95 latency for the affected endpoint.

Ask the developer which endpoint or operation to baseline before querying.

### Layer 2 — APM manual collection

Ask the developer to open their APM (New Relic, Datadog, Scout, Skylight) and provide:

- P50 and P95 response time for the affected endpoint (last 7 days)
- Top 5 slowest SQL queries (query text + average duration)
- Memory per dyno: average and max (last 24 hours)
- Throughput: requests per minute

### Layer 1 — Local profiling tools

**Request profiling with rack-mini-profiler:**

```ruby
# Gemfile (development group)
gem 'rack-mini-profiler'
gem 'flamegraph'   # for flamegraph support
gem 'stackprof'    # for flamegraph support
```

Visit the slow endpoint in development. rack-mini-profiler shows:
- Total request time (ms)
- SQL query count
- SQL total time (ms)

**Memory profiling:**

```ruby
# Gemfile (development group)
gem 'memory_profiler'
```

```ruby
# In rails console or a benchmark script
require 'memory_profiler'
report = MemoryProfiler.report { MyClass.heavy_operation }
report.pretty_print
# Key numbers: total_allocated_memsize, total_retained_memsize
```

**Database query profiling:**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ...;
-- Key numbers: actual time, rows, Planning Time, Execution Time
```

**Benchmark comparison:**

```ruby
# Gemfile (development group)
gem 'benchmark-ips'
```

```ruby
require 'benchmark/ips'
Benchmark.ips do |x|
  x.report('operation') { MyClass.heavy_operation }
  x.compare!
end
# Key number: iterations per second
```

## Step 2: Record the baseline

Document exactly — copy this template into the conversation or PR description:

```
Baseline recorded: [YYYY-MM-DD]
Endpoint/operation: [name]
Measurement source: [APM name / rack-mini-profiler / benchmark-ips / EXPLAIN ANALYZE]

Metrics:
- Response time P95:    [X ms]
- SQL query count:      [N]
- SQL total time:       [X ms]
- Memory allocated:     [X MB]
- Memory retained:      [X MB]
- [other relevant metric]: [value]
```

## Step 3: Proceed

With baseline recorded, invoke `systematic-diagnosis`.
