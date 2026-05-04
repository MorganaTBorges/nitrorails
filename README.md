# nitrorails

A disciplined performance optimization copilot for senior Rails developers.

## How it works

nitrorails enforces one rule above all others: **never optimize without measuring first.**

When you bring a performance problem to your coding agent, nitrorails doesn't let it guess. It steps back, establishes a baseline, and identifies the real bottleneck before touching a single line of code. Once the optimization is done, it forces a comparison against the baseline before you can call it complete.

The cycle is always the same: **measure → diagnose → optimize → verify.**

Because skills trigger automatically, you don't need to do anything special. Your coding agent just has nitrorails.

## Why nitrorails?

Performance work fails in predictable ways:

- Optimizing the wrong layer (Ruby code when the problem is missing indexes)
- Shipping changes with no before/after proof they worked
- Caching things that don't need caching
- Fixing N+1s in the wrong direction (`includes` when `joins` is correct)

nitrorails is a set of enforced checkpoints that prevent each of these mistakes. It won't explain what an N+1 is — it assumes you already know. What it does is make sure you follow the right process every time.

## Installation

### Claude Code

```bash
/plugin install nitrorails@nitrorails
```

### Claude Code (via GitHub)

```bash
/plugin marketplace add MorganaTBorges/nitrorails
/plugin install nitrorails@nitrorails
```

### Verify installation

Start a new session and mention a performance problem. The agent should automatically invoke `profiling-first` before any other action.

## The optimization workflow

1. **profiling-first** — Establishes a measurable baseline before any work begins. Hard gate: no diagnosis without numbers.

2. **systematic-diagnosis** — Identifies the real bottleneck (database, memory, CPU, I/O, or Ruby code) before any code changes.

3. **optimization-workflow** — Enforces the cycle: baseline → hypothesis → change → measure → document result.

4. **verification-after-optimization** — Compares against the baseline. Hard gate: no PR without a recorded "after" number.

**The agent checks for relevant skills before any task. These are enforced checkpoints, not suggestions.**

## Skills catalog

| Type    | Skill                             | When to invoke                                        |
|---------|-----------------------------------|-------------------------------------------------------|
| Process | `using-nitrorails`                | Every conversation start                              |
| Process | `profiling-first`                 | Before any optimization                               |
| Process | `systematic-diagnosis`            | Before any code change, to identify the real bottleneck |
| Process | `optimization-workflow`           | When implementing an optimization                     |
| Process | `verification-after-optimization` | Before declaring an optimization done                 |
| Domain  | `activerecord-performance`        | N+1, query issues, bulk operations                    |
| Domain  | `caching-strategies`              | Fragment, HTTP, low-level, Redis caching              |
| Domain  | `database-optimization`           | Indexes, query plans, connection pooling              |
| Domain  | `background-jobs-performance`     | Job design, memory bloat, concurrency                 |
| Domain  | `memory-optimization`             | Leaks, GC pressure, allocation, jemalloc              |
| Domain  | `asset-performance`               | Bundle size, CDN, lazy loading                        |

## Measuring performance

nitrorails supports three measurement layers in order of preference:

1. **Local tools** — rack-mini-profiler output, `benchmark-ips` results, `EXPLAIN ANALYZE`. Paste the output and the agent builds the baseline from it. Works offline, no configuration required.

2. **APM manual collection** — The agent asks for specific metrics from your APM (New Relic, Datadog, Scout, Skylight): P95 response time, top slow queries, memory per dyno. You paste the numbers; the agent stores them as the baseline.

3. **MCP integration** — If you have a New Relic or Datadog MCP server configured, the agent queries it directly without manual intervention.

## What nitrorails does not cover

- Frontend performance beyond Rails asset serving (React, Vue, SPA optimization)
- Infrastructure optimization (server sizing, Kubernetes, load balancing)
- Database administration beyond what a Rails developer controls
- Tutorial-level explanations of Rails fundamentals

## Contributing

Skills live directly in this repository.

1. Fork the repository
2. Create a branch for your skill or fix
3. Submit a PR with a description of what the skill does and when it triggers

Found incorrect information or have a suggestion? Open an issue at https://github.com/MorganaTBorges/nitrorails/issues

## License

MIT License — see LICENSE for details.
