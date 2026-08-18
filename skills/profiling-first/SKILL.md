---
name: profiling-first
description: Use before any Rails performance optimization — establishes a measurable baseline using local tools, APM manual collection, or MCP integration. Hard gate: no diagnosis or optimization without a recorded baseline.
---

<HARD-GATE>
Do NOT proceed to diagnosis or optimization without a recorded baseline.
If the developer has not provided a baseline, complete this skill first.
A baseline is a pair of numbers (BEFORE and NOW), or an explicit list of the
sources that failed to produce one. A single "current" number is not a baseline.
</HARD-GATE>

## Goal

Record a measurable baseline before any code changes. This baseline is the "before" number that `verification-after-optimization` will compare against.

## Step 0: Establish what you are baselining

A ticket describes a problem observed at some point in the past. The baseline is
not "the current number" — it is the pair:

- **BEFORE** — the metric when the problem was observed
- **NOW** — the metric today

Ask yourself which case you are in:

| Case | BEFORE comes from |
|---|---|
| You can reproduce the problem right now (locally or in production) | Today's measurement of the unpatched code. NOW and BEFORE are the same number. |
| The problem was reported days or weeks ago and you cannot reproduce it | Your APM's historical data. This is the case that traps people — see Step 1. |

If NOW is healthy and you never obtained BEFORE, you have not measured anything.
You have only observed that the page is fast today, which does not tell you
whether it was ever slow or what made it fast.

## Step 1: Choose the measurement layer

Work from highest to lowest availability:

### Layer 3 — APM query access (MCP or API, if configured)

Every APM stores performance data in two places, with very different retention:

| Store | Contains | Retention |
|---|---|---|
| Per-request events / traces / samples | one row per request, full attributes | short (days) |
| Pre-aggregated metrics / rollups | averages per endpoint per interval | long (months) |

**An old ticket's BEFORE number lives in the second store.** Querying the first
one for a window that has expired returns zero rows, which reads exactly like
"this endpoint has no data" and is the most common way a baseline gets abandoned.
An empty result from the event store means WRONG STORE, not "unavailable".

Look up your APM's names for the two stores before you start. Examples:

- **New Relic** — events: `Transaction` (NRQL, ~8 days by default, plan-dependent).
  Rollups: `Metric` with `newrelic.timeslice.value`, filtered by
  `metricTimesliceName`, retained far longer. Use `TIMESERIES 1 week SINCE 90 days ago`.
- **Datadog** — events: spans/traces. Rollups: trace metrics (`trace.*.request`),
  retained much longer than the spans they came from.
- Scout, Skylight, AppSignal, Sentry — the same split under different names. Check
  the vendor's retention docs; the aggregate store is always the long one.

Read the ticket and the git history of the affected files before querying, so the
series can be lined up against deploy dates. Ask the developer which endpoint to
baseline only if the ticket does not name one.

### Layer 2 — APM manual collection

Ask the developer to open their APM (New Relic, Datadog, Scout, Skylight) and provide:

- P50 and P95 response time for the affected endpoint (last 7 days)
- The same two numbers for the window when the problem was reported
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

## Never conclude "the data is unavailable"

Before writing that a number cannot be obtained, you MUST have tried a second
source, and you MUST name the sources that failed.

| First attempt found nothing | Try next |
|---|---|
| Event/trace store, window expired | The aggregate/rollup store (see Layer 3) |
| No APM data for this endpoint at all | Application logs, or reproduce locally (Layer 1) |
| Endpoint name not found | List the recorded endpoint names and match on a fragment |
| Cannot reproduce locally | Ask the developer for the conditions: which user, how much data, which locale |

"Probably", "likely" and "presumably" inside a baseline mean you stopped early.
A baseline is a number, or an explicit list of the sources that failed to yield one.

## Step 2: Record the baseline

Document exactly — copy this template into the conversation or PR description:

```
Baseline recorded: [YYYY-MM-DD]
Endpoint/operation: [name]
Measurement source: [APM name + exact query / rack-mini-profiler / benchmark-ips / EXPLAIN ANALYZE]

BEFORE (window the problem was reported: [dates])
- Response time avg / P95:  [X ms]
- SQL query count:          [N]
- SQL total time:           [X ms]
- Memory allocated:         [X MB]
- Memory retained:          [X MB]
- Sample size:              [N requests]

NOW
- Response time avg / P95:  [X ms]
- SQL query count:          [N]
- SQL total time:           [X ms]
- Memory allocated:         [X MB]
- Memory retained:          [X MB]
- Sample size:              [N requests]

Delta: [X% better / worse]
Deploys inside the window: [sha + date + title, or "none"]
```

Omit rows that do not apply to the bottleneck being measured. Do NOT omit either
block: if one of them is empty, the gate has not opened. Say which source failed
and stop.

## Step 3: Proceed, or stop

Compare BEFORE and NOW before choosing the next step:

| Baseline says | Next |
|---|---|
| Problem confirmed and still happening | Invoke `systematic-diagnosis` |
| Problem is gone | **STOP.** Report the delta, identify the change that likely fixed it, hand the ticket back. Do NOT invoke `systematic-diagnosis` — optimizing something already fast is speculative work, which is exactly what this skill exists to prevent. |
| Problem is real but in a different endpoint or layer than the ticket claims | Report the mismatch and get the scope corrected first |

Residual inefficiency noticed while measuring is NOT a reason to continue. Record
it as a note and let the developer decide whether it earns its own baseline.
