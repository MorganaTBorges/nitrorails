# Roadmap

Work planned for nitrorails, roughly in priority order.

## New domain skills

- [ ] `web-server-concurrency`: Puma workers/threads, connection pool sizing, request queueing time.
- [ ] `view-rendering-performance`: partial and collection rendering, ViewComponent/Phlex cost, template caching.
- [ ] `solid-stack-performance`: Solid Queue, Solid Cache and Solid Cable tuning on Rails 8.
- [ ] `api-performance`: serializer cost, pagination, payload size, HTTP caching for JSON APIs.

## Existing skills

- [ ] Add a worked example (before/after numbers) to every domain skill.
- [ ] Cover Rails 8.1 changes in `activerecord-performance` and `caching-strategies`.
- [ ] Add a tool matrix to `profiling-first` (rack-mini-profiler, stackprof, vernier, memory_profiler, APMs).

## Quality

- [ ] Example Rails app with seeded performance problems (N+1, missing index, memory bloat) to exercise the skills end to end.
- [ ] CI check that validates every `SKILL.md` frontmatter and the plugin/marketplace manifests.
- [ ] Shell tests for the `session-start` hook.

## Docs and distribution

- [ ] Per-skill docs page with when it triggers and what it outputs.
- [ ] Install instructions for other coding agents besides Claude Code.
- [ ] CHANGELOG and tagged GitHub releases.
