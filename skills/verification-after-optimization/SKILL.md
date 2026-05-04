---
name: verification-after-optimization
description: Use before marking any Rails performance optimization as complete — forces baseline comparison, regression check, and production-like validation. Hard gate: no PR without a recorded "after" number.
---

<HARD-GATE>
Do NOT open a PR or mark this task done without completing every item in this checklist.
A performance optimization without measured proof is not an optimization.
</HARD-GATE>

## Verification Checklist

### 1. Baseline comparison
- [ ] "Before" metrics recorded from `profiling-first`
- [ ] "After" metrics recorded using the same method and environment
- [ ] Percentage improvement calculated for each metric
- [ ] Improvement is ≥ 20% (if less, document why it's still worth shipping)

### 2. Regression check
- [ ] Test suite passes with no new failures
- [ ] Bullet gem still silent on the affected endpoint (no new N+1 introduced)
- [ ] Adjacent endpoints tested — no regressions in response time or query count
- [ ] Memory usage did not increase in adjacent operations
- [ ] If caching was added: verified cache invalidation works correctly

### 3. Production-like validation
- [ ] Measurement taken on staging (not just development)
- [ ] Data volume on staging is representative of production
- [ ] If fragment cache added: cache hit rate is > 80% for warm traffic
- [ ] If index added: `EXPLAIN ANALYZE` confirms the index is used (`Index Scan`, not `Seq Scan`)
- [ ] If background job changed: job processes correctly end-to-end in staging

### 4. PR readiness
- [ ] PR description includes the before/after table (template below)
- [ ] Change is documented with the measurement method used
- [ ] Change is reversible via a one-line revert or a follow-up migration

## PR Description Template

```markdown
## Performance optimization: [endpoint or operation name]

**Problem:** [describe the symptom — P95 was X ms, memory was growing, etc.]

**Root cause identified by:** systematic-diagnosis — [database N+1 / missing index / memory leak / etc.]

**Change:** [one sentence describing what was changed]

**Results:**

| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| P95 response time | X ms | Y ms | -Z% |
| SQL query count | N | M | -K queries |
| SQL total time | X ms | Y ms | -Z% |
| Memory per request | X MB | Y MB | -Z% |

**Measurement method:** [rack-mini-profiler / APM name / benchmark-ips / EXPLAIN ANALYZE]
**Environment:** [staging / development — note data volume if relevant]
```
