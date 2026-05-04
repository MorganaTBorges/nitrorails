---
name: memory-optimization
description: Use when diagnosing or reducing Rails application memory usage — object allocation tracking, GC pressure, memory leaks in long-running processes, derailed_benchmarks, jemalloc, closures retaining objects.
---

## Measuring Memory

### derailed_benchmarks — boot and request memory

```ruby
# Gemfile (development group)
gem 'derailed_benchmarks', require: false
gem 'stackprof', require: false
```

```bash
# How much memory does booting the app use?
bundle exec derailed bundle:mem

# How much memory does a single request allocate?
bundle exec derailed exec perf:mem

# Object allocation per request (with stackprof)
bundle exec derailed exec perf:objects

# Memory growth over N requests (leak detection)
bundle exec derailed exec perf:mem_over_time
```

`perf:mem_over_time` is the most useful for finding leaks — memory should stabilize after warm-up, not keep growing.

### memory_profiler — allocation by source

```ruby
require 'memory_profiler'
report = MemoryProfiler.report { [reproduce the operation] }
report.pretty_print

# Key sections:
# "allocated memory by gem"   — which gem allocates the most
# "allocated memory by file"  — which file allocates the most
# "retained memory by class"  — what is not GC'd
```

Retained memory is a sign of a leak. Allocated memory that is collected is normal.

## Common Memory Leak Patterns

### Class-level mutable state

```ruby
# Bad: @cache grows forever — never GC'd because it's on the class
class PostService
  @cache = {}
  def self.find(id)
    @cache[id] ||= Post.find(id)
  end
end

# Good: use Rails.cache with expiration instead
class PostService
  def self.find(id)
    Rails.cache.fetch("post/#{id}", expires_in: 5.minutes) { Post.find(id) }
  end
end
```

### Closures capturing large objects

```ruby
# Bad: the proc captures the entire `data` hash even if only `id` is needed
data = { id: 1, payload: large_string }
callback = -> { process(data[:id]) }

# Good: capture only what you need
id = data[:id]
callback = -> { process(id) }
```

### Growing arrays or hashes across requests

```ruby
# Bad: accumulated across the process lifetime if @events is a class variable
class EventTracker
  @events = []
  def self.track(event)
    @events << event  # grows forever
  end
end
```

### Memoization without bounds

```ruby
# Bad: memoization cache grows as more users are processed
def expensive_for(user_id)
  @cache ||= {}
  @cache[user_id] ||= compute(user_id)
end
```

## GC Tuning

Ruby's GC is conservative by default. These environment variables can reduce GC pressure:

```bash
# Increase the heap growth factor — fewer GC runs, more memory used
RUBY_GC_HEAP_GROWTH_FACTOR=1.25

# Start with a larger heap — avoids GC during warm-up
RUBY_GC_HEAP_INIT_SLOTS=600000

# Minimum oldgen size before major GC
RUBY_GC_OLDMALLOC_LIMIT=64000000
```

Profile GC behavior:
```ruby
GC.stat  # returns a hash with :heap_live_slots, :minor_gc_count, :major_gc_count
```

High `major_gc_count` relative to `minor_gc_count` suggests many long-lived objects surviving into oldgen.

## jemalloc

Replace Ruby's default allocator with jemalloc to reduce memory fragmentation. Most effective on Heroku and similar platforms where Ruby runs for long periods.

```dockerfile
# Dockerfile / Heroku
RUN apt-get install -y libjemalloc2
ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
```

For Heroku, add to `Procfile`:
```
web: LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 bundle exec puma -C config/puma.rb
```

Expected improvement: 10-30% memory reduction on long-running processes. Verify with `derailed exec perf:mem_over_time` before and after.

## Reducing Boot Memory

```bash
bundle exec derailed bundle:mem
```

Identifies the heaviest gems at boot. Common fixes:
- Move heavy gems to `require: false` and require them lazily
- Remove unused gems
- Replace heavy gems with lighter alternatives (e.g., `faraday` -> `httpx`)

```ruby
# Gemfile — lazy require
gem 'aws-sdk-s3', require: false  # require 'aws-sdk-s3' only where used
```
