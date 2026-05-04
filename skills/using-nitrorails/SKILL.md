---
name: using-nitrorails
description: Use when starting any conversation about Rails performance — presents the optimization cycle, the skills catalog, and enforces the core rule that no optimization proceeds without a baseline measurement
---

<EXTREMELY-IMPORTANT>
Never optimize without measuring first.

If a performance problem is mentioned, invoke profiling-first BEFORE any code analysis, suggestions, or optimizations.

This is not optional.
</EXTREMELY-IMPORTANT>

## The nitrorails Optimization Cycle

Every performance task follows this cycle without exception:

1. **Measure** → `profiling-first` — establish a baseline before touching anything
2. **Diagnose** → `systematic-diagnosis` — identify the real bottleneck layer
3. **Optimize** → invoke the relevant domain skill + `optimization-workflow`
4. **Verify** → `verification-after-optimization` — prove the improvement before shipping

## Skills Catalog

| Type    | Skill                             | When to invoke                                          |
|---------|-----------------------------------|---------------------------------------------------------|
| Process | `profiling-first`                 | Before any optimization                                 |
| Process | `systematic-diagnosis`            | Before any code change, to identify the real bottleneck |
| Process | `optimization-workflow`           | When implementing an optimization                       |
| Process | `verification-after-optimization` | Before declaring an optimization done                   |
| Domain  | `activerecord-performance`        | N+1, query issues, bulk operations                      |
| Domain  | `caching-strategies`              | Fragment, HTTP, low-level, Redis caching                |
| Domain  | `database-optimization`           | Indexes, query plans, connection pooling                |
| Domain  | `background-jobs-performance`     | Job design, memory bloat, concurrency                   |
| Domain  | `memory-optimization`             | Leaks, GC pressure, object allocation, jemalloc         |
| Domain  | `asset-performance`               | Bundle size, CDN, lazy loading                          |

## Starting a Performance Session

When a developer brings a performance problem:

1. Ask: "Do you have a baseline measurement?" If no — invoke `profiling-first` immediately.
2. Once baseline is established — invoke `systematic-diagnosis`.
3. Once bottleneck layer is identified — invoke the matching domain skill.
4. As optimization is implemented — follow `optimization-workflow`.
5. Before opening a PR — invoke `verification-after-optimization`.

Never skip steps. Never optimize based on intuition alone.
