# nitrorails

nitrorails is a complete performance optimization workflow for your Rails coding agent, built on top of a set of composable skills that enforce the right process every time.

## How it works

It starts the moment you mention a performance problem. Instead of guessing and jumping into code changes, your agent steps back and asks: do you have a baseline? It establishes one first, identifies the real bottleneck, then guides the optimization — and won't let you ship without proof it worked.

The cycle is always the same: **measure → diagnose → optimize → verify.**

Because skills trigger automatically, you don't need to do anything special. Your coding agent just has nitrorails.

## Sources

The skills in nitrorails are built on knowledge from:

- [The Complete Guide to Rails Performance](https://www.railsspeed.com) by Nate Berkopec
- [Ruby Performance Optimization](https://pragprog.com/titles/adrpo/ruby-performance-optimization/) by Alexander Dymo
- [rack-mini-profiler](https://github.com/MiniProfiler/rack-mini-profiler), [derailed_benchmarks](https://github.com/zombocom/derailed_benchmarks), [Bullet](https://github.com/flyerhzm/bullet) documentation
- Performance engineering practices from Shopify, GitHub, and Basecamp engineering blogs

## Installation

**Note:** Installation differs by platform. Claude Code and Cursor have built-in plugin marketplaces. Codex and OpenCode require manual setup.

### Claude Code — Official Marketplace

```bash
/plugin install nitrorails@nitrorails
```

### Claude Code — via GitHub

```bash
/plugin marketplace add MorganaTBorges/nitrorails
/plugin install nitrorails@nitrorails
```

### Cursor

```bash
cd ~/.cursor/plugins/local/ && git clone https://github.com/MorganaTBorges/nitrorails
```

### Codex

Tell Codex:

```
Fetch and follow instructions from https://raw.githubusercontent.com/MorganaTBorges/nitrorails/main/README.md
```

### GitHub Copilot CLI

```bash
copilot plugin marketplace add MorganaTBorges/nitrorails
copilot plugin install nitrorails@nitrorails
```

### Gemini CLI

```bash
gemini extensions install https://github.com/MorganaTBorges/nitrorails
```

### Verify installation

Start a new session and mention a performance problem. The agent should automatically invoke `profiling-first` before suggesting any code changes.

## The optimization workflow

1. **profiling-first** — Establishes a measurable baseline before any work begins. Hard gate: no diagnosis without numbers.

2. **systematic-diagnosis** — Identifies the real bottleneck (database, memory, CPU, I/O, or Ruby code) before any code changes.

3. **optimization-workflow** — Enforces the cycle: baseline → hypothesis → change → measure → document result.

4. **verification-after-optimization** — Compares against the baseline. Hard gate: no PR without a recorded "after" number.

**The agent checks for relevant skills before any task. These are enforced checkpoints, not suggestions.**

## Skills catalog

### Process skills

| Skill | When to invoke |
|-------|---------------|
| `using-nitrorails` | Every conversation start |
| `profiling-first` | Before any optimization |
| `systematic-diagnosis` | Before any code change, to identify the real bottleneck |
| `optimization-workflow` | When implementing an optimization |
| `verification-after-optimization` | Before declaring an optimization done |

### Domain skills

| Skill | When to invoke |
|-------|---------------|
| `activerecord-performance` | N+1, query issues, bulk operations |
| `caching-strategies` | Fragment, HTTP, low-level, Redis caching |
| `database-optimization` | Indexes, query plans, connection pooling |
| `background-jobs-performance` | Job design, memory bloat, concurrency |
| `memory-optimization` | Leaks, GC pressure, allocation, jemalloc |
| `asset-performance` | Bundle size, CDN, lazy loading |

## Measuring performance

nitrorails supports three measurement layers in order of preference:

1. **Local tools** — rack-mini-profiler output, `benchmark-ips` results, `EXPLAIN ANALYZE`. Paste the output and the agent builds the baseline from it. Works offline, no configuration required.

2. **APM manual collection** — The agent asks for specific metrics from your APM (New Relic, Datadog, Scout, Skylight): P95 response time, top slow queries, memory per dyno. You paste the numbers; the agent stores them as the baseline.

3. **MCP integration** — If you have a New Relic or Datadog MCP server configured, the agent queries it directly without manual intervention.

## Philosophy

- **Measure before optimizing** — Numbers first, always. Intuition is a starting point for a hypothesis, not a justification for shipping.
- **One change at a time** — If you change three things at once, you can't know which one worked.
- **Evidence over claims** — An optimization without a before/after comparison is a guess.
- **Systematic over ad-hoc** — The same process every time, regardless of the problem.

## What nitrorails does not cover

- Frontend performance beyond Rails asset serving (React, Vue, SPA optimization)
- Infrastructure optimization (server sizing, Kubernetes, load balancing)
- Database administration beyond what a Rails developer controls

## Contributing

Please read our [Code of Conduct](CODE_OF_CONDUCT.md) before contributing.

Skills live directly in this repository. To contribute:

1. Fork the repository
2. Create a branch for your skill or fix
3. Submit a PR with a description of what the skill does and when it triggers

Found incorrect information or have a suggestion? [Open an issue](https://github.com/MorganaTBorges/nitrorails/issues).

## Updating

Skills update automatically when you update the plugin:

```bash
/plugin update nitrorails
```

## License

MIT License — see LICENSE for details.
