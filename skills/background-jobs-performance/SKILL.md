---
name: background-jobs-performance
description: Use when optimizing background jobs — job design to prevent memory bloat, batching large datasets, eliminating N+1 inside jobs, Sidekiq and Delayed Job concurrency tuning, idempotent job design, queue monitoring.
---

## Memory Bloat in Jobs

Long-running jobs accumulate memory because:
- Loading large ActiveRecord collections into memory
- Holding references to objects that cannot be GC'd
- Gems with memory leaks accumulating over many iterations

### Rule: never load all records at once

```ruby
# Bad: loads 500k records into memory for the entire job duration
class DigestJob < ApplicationJob
  def perform
    User.all.each { |u| DigestMailer.weekly(u).deliver_later }
  end
end

# Good: process in batches — memory is released between batches
class DigestJob < ApplicationJob
  def perform
    User.find_each(batch_size: 500) { |u| DigestMailer.weekly(u).deliver_later }
  end
end
```

### Avoid accumulating objects in instance variables

```ruby
# Bad: @results grows unboundedly
class ReportJob < ApplicationJob
  def perform
    @results = []
    Record.find_each { |r| @results << process(r) }
    save_results(@results)  # now holds all results in memory
  end
end

# Good: write incrementally
class ReportJob < ApplicationJob
  def perform
    CSV.open('tmp/report.csv', 'w') do |csv|
      Record.find_each { |r| csv << process(r) }
    end
    upload_and_cleanup
  end
end
```

## N+1 Inside Jobs

Jobs have no HTTP request cycle to tell you about N+1 — they're silent. Use the same tools:

```ruby
# Bad: N+1 inside a job
class PublishJob < ApplicationJob
  def perform(post_ids)
    Post.where(id: post_ids).each do |post|
      post.author.notify_followers  # N query for author, N query for followers
    end
  end
end

# Good: preload associations
class PublishJob < ApplicationJob
  def perform(post_ids)
    Post.where(id: post_ids).includes(author: :followers).each do |post|
      post.author.notify_followers
    end
  end
end
```

Enable Bullet in test environment to catch N+1 in job specs:
```ruby
# config/environments/test.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.raise  = true
end
```

## Batching Large Operations

### Using job-iteration gem (Shopify)

Handles interruptions, deploys mid-run, and memory limits safely:

```ruby
# Gemfile
gem 'job-iteration'

class MigrationJob < ApplicationJob
  include JobIteration::Iteration

  def build_enumerator(cursor:)
    enumerator_builder.active_record_on_records(
      Post.where(migrated: false).order(:id),
      cursor:
    )
  end

  def each_iteration(post)
    post.migrate!
  end
end
```

### Manual batching with Sidekiq

Enqueue individual jobs per record instead of processing all records in one job:

```ruby
# Enqueue fan-out
class FanOutJob < ApplicationJob
  def perform
    User.find_each(batch_size: 500) do |user|
      ProcessUserJob.perform_later(user.id)
    end
  end
end

# Process one at a time — horizontally scalable, no memory bloat
class ProcessUserJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    # process one user
  end
end
```

## Sidekiq Concurrency Tuning

```yaml
# config/sidekiq.yml
:concurrency: 10  # default — one thread per concurrent job

# Rule: concurrency <= database connection pool size
# If concurrency: 10, database.yml pool must be >= 10
```

High concurrency means more parallel DB connections. If your DB pool is 5 and Sidekiq has 10 threads, 5 threads will wait.

For memory-intensive jobs, reduce concurrency or use a dedicated queue:
```yaml
:queues:
  - [critical, 3]
  - [default, 2]
  - [memory_intensive, 1]  # one at a time for heavy jobs
```

## Delayed Job Tuning

```ruby
# config/initializers/delayed_job.rb
Delayed::Worker.max_attempts        = 3
Delayed::Worker.max_run_time        = 10.minutes
Delayed::Worker.sleep_delay         = 5       # seconds between polls when queue is empty
Delayed::Worker.destroy_failed_jobs = false   # keep failed jobs for inspection
```

For large tables, add indexes:
```ruby
# migration
add_index :delayed_jobs, [:priority, :run_at], name: 'delayed_jobs_priority'
add_index :delayed_jobs, :locked_by
```

Clean up old jobs regularly:
```ruby
# In a scheduled job or rake task
Delayed::Job.where('created_at < ? AND last_error IS NULL', 7.days.ago).delete_all
```

## Idempotent Jobs

Jobs can run more than once (retries, at-least-once delivery). Design them to be safe to repeat:

```ruby
# Bad: creates a duplicate record on retry
class CreateReportJob < ApplicationJob
  def perform(user_id, date)
    Report.create!(user_id: user_id, date: date, data: compute(user_id, date))
  end
end

# Good: idempotent with find_or_initialize_by
class CreateReportJob < ApplicationJob
  def perform(user_id, date)
    report = Report.find_or_initialize_by(user_id: user_id, date: date)
    report.data = compute(user_id, date)
    report.save!
  end
end
```

## Queue Monitoring

Check queue depth regularly:
```ruby
# Sidekiq
Sidekiq::Queue.new('default').size      # jobs waiting
Sidekiq::Queue.new('default').latency   # seconds since oldest job was enqueued
Sidekiq::Workers.new.size               # running right now
Sidekiq::RetrySet.new.size              # waiting to retry

# Delayed Job
Delayed::Job.where(failed_at: nil, locked_at: nil).count  # pending
Delayed::Job.where.not(failed_at: nil).count              # failed
```

Alert when `queue.latency > 60` (jobs waiting more than 60 seconds).
