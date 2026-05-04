---
name: activerecord-performance
description: Use when optimizing ActiveRecord queries — N+1 detection and resolution, association loading strategy (includes/joins/preload/eager_load), counter_cache, find_each, bulk operations, strict_loading. Covers the majority of Rails performance problems.
---

## Association Loading Strategy

The four methods are not interchangeable. Use the wrong one and you either generate a JOIN you don't need or load records you won't use.

| Method | SQL generated | Use when |
|--------|--------------|----------|
| `preload` | 2 separate queries | You'll read the association, no WHERE/ORDER on it |
| `eager_load` | LEFT OUTER JOIN | You need WHERE or ORDER on the association |
| `includes` | 2 queries OR JOIN (Rails decides) | General use — Rails picks the strategy |
| `joins` | INNER JOIN, no association loaded | You filter on the association but don't read its data |

```ruby
# N+1 — one query per post
Post.limit(10).each { |p| p.author.name }

# Fixed with preload — 2 queries total
Post.preload(:author).limit(10).each { |p| p.author.name }

# Filter on association — use joins (data not loaded)
Post.joins(:author).where(authors: { verified: true })

# Filter AND read — use eager_load (one JOIN query)
Post.eager_load(:author).where(authors: { verified: true }).map { |p| p.author.name }
```

Nested associations:
```ruby
Post.includes(comments: :author).limit(10)
Post.includes(:tags, author: :avatar).limit(10)
```

## Avoid SELECT *

```ruby
# Bad: loads all columns, including large text/jsonb fields
Post.all.map(&:title)

# Good: load only what you need
Post.select(:id, :title, :published_at).all

# Good: pluck returns plain values, no model instantiation
Post.where(published: true).pluck(:id, :title)

# Good: pick returns the first row as an array
Post.where(id: 1).pick(:title, :body)
```

## counter_cache

Eliminates COUNT queries on `has_many` associations:

```ruby
# 1. Migration
add_column :posts, :comments_count, :integer, default: 0, null: false
# Reset if table already has data:
# Post.find_each { |p| Post.reset_counters(p.id, :comments) }

# 2. Model
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# 3. Usage — no SQL query, reads from the cached column
post.comments.size   # uses counter_cache
post.comments.count  # always hits the DB — avoid this
```

## Large Dataset Iteration

Never `.all.each` on a large table — it loads every record into memory at once.

```ruby
# Bad: loads 500k records into memory
User.all.each { |u| u.send_digest_email }

# Good: processes in batches of 1000 (default)
User.find_each { |u| u.send_digest_email }

# Good: with conditions and custom batch size
User.where(digest_enabled: true).find_each(batch_size: 500) do |u|
  u.send_digest_email
end

# Good: when you need the batch as a collection
User.find_in_batches(batch_size: 500) do |users|
  DigestMailer.batch(users).deliver_later
end
```

## Bulk Operations

```ruby
# Bad: N INSERT statements
records.each { |attrs| Model.create!(attrs) }

# Good: single INSERT statement (Rails 6+)
Model.insert_all(records)           # ignores duplicates
Model.upsert_all(records, unique_by: :email)  # upserts on conflict

# Bad: N UPDATE statements
users.each { |u| u.update!(last_seen_at: Time.current) }

# Good: single UPDATE
User.where(id: user_ids).update_all(last_seen_at: Time.current)

# Bad: N DELETE statements
old_records.each(&:destroy)

# Good: single DELETE
Model.where('created_at < ?', 90.days.ago).delete_all
# Note: delete_all skips callbacks and validations
```

## Useful Query Methods

```ruby
# exists? is faster than count > 0 for presence checks
Post.where(author: user).exists?   # SELECT 1 LIMIT 1
Post.where(author: user).any?      # same as exists?

# size vs count on associations
post.comments.loaded? ? post.comments.size : post.comments.count

# Avoid loading records just to check if a scope is empty
Post.published.none?   # SELECT 1 LIMIT 1
Post.published.empty?  # forces load if not loaded — avoid
```

## strict_loading

Raises `ActiveRecord::StrictLoadingViolationError` when lazy-loading an association that wasn't preloaded. Use in development and test to catch N+1 at the source.

```ruby
# Per query
Post.strict_loading.limit(10).each { |p| p.author.name }
# => raises if author wasn't preloaded

# Per model (development and test environments)
class ApplicationRecord < ActiveRecord::Base
  self.strict_loading_by_default = !Rails.env.production?
end
```

## Detecting N+1 with Bullet

```ruby
# Gemfile (development, test)
gem 'bullet'

# config/environments/development.rb
config.after_initialize do
  Bullet.enable       = true
  Bullet.alert        = true
  Bullet.rails_logger = true
  Bullet.add_footer   = true
  Bullet.raise        = true  # raises in test — catches N+1 in CI
end

# spec/rails_helper.rb (optional)
config.before(:each) { Bullet.start_request }
config.after(:each)  { Bullet.end_request }
```
