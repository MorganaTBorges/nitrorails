---
name: optimization-workflow
description: Use when implementing any Rails performance optimization — enforces the baseline → hypothesis → change → measure → document cycle for all bottleneck types
---

## Goal

Every optimization must be intentional and measured. Intuition is a starting point for a hypothesis, not a justification for shipping.

## The Cycle

### Step 1: State the hypothesis

Before writing any code, state explicitly in the conversation:

```
Hypothesis:           [what I believe is causing the problem]
Expected improvement: [metric from X to Y — e.g., P95 from 800ms to 200ms]
Approach:             [what will change and why this addresses the root cause]
Reversibility:        [how to undo this change if it causes regressions]
```

A hypothesis with no expected improvement is a guess. Quantify it.

### Step 2: Make the smallest change that tests the hypothesis

One change at a time. If you change the query strategy AND add an index AND add caching in the same commit, you cannot identify which change produced the improvement.

Invoke the matching domain skill for the actual implementation:
- `activerecord-performance` — ORM query changes
- `caching-strategies` — adding or tuning cache
- `database-optimization` — index, query plan, connection pool
- `background-jobs-performance` — job design, batching
- `memory-optimization` — allocation, GC, jemalloc
- `asset-performance` — bundle, CDN, compression

### Step 3: Measure with the same method as the baseline

Use the exact same tool and conditions as `profiling-first`:
- Same endpoint, same operation
- Staging environment preferred over development
- Same data volume (development DBs are often much smaller than production)

Record the new metrics using the same template from `profiling-first`.

### Step 4: Compare and document

```
Optimization result:
Change made:           [description of the change]
Hypothesis confirmed:  [yes / no / partially]

Before:
  - Response time P95: X ms
  - SQL query count:   N
  - Memory allocated:  X MB

After:
  - Response time P95: Y ms
  - SQL query count:   M
  - Memory allocated:  Y MB

Delta:                 [Z% improvement]
Notes:                 [anything unexpected — e.g., cache warm-up, staging vs prod differences]
```

### Step 5: Decide next action

**Hypothesis confirmed, improvement significant (≥ 20%):**
→ Proceed to `verification-after-optimization`.

**Hypothesis not confirmed (no measurable improvement):**
→ Return to `systematic-diagnosis`. The bottleneck layer may have been misidentified.

**Improvement marginal (< 10%):**
→ Evaluate whether the added complexity is justified. A 5% improvement from adding a cache layer may not be worth the operational overhead.

**Improvement confirmed but regressions detected:**
→ Fix regressions before proceeding. Do not ship an optimization that breaks something else.
