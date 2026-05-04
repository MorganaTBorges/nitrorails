---
name: caching-strategies
description: Use when adding or tuning Rails caching — fragment caching, Russian doll caching, low-level cache, HTTP caching (stale?/ETags), cache key design, expiration, Redis configuration. Includes guidance on when NOT to cache.
---

## When NOT to cache

Cache introduces complexity: stale data, invalidation bugs, cold-start latency, and memory pressure. Before adding cache, verify:

- The bottleneck is confirmed by `systematic-diagnosis` — cache only if CPU/rendering time is the issue, not a missing index
- The data changes infrequently enough that stale reads are acceptable
- The computation being cached is genuinely expensive (> 50ms)

## Fragment Caching

Cache rendered HTML fragments. Best for view partials that are expensive to render.

```erb
<%# app/views/posts/_post.html.erb %>
<% cache post do %>
  <article>
    <h2><%= post.title %></h2>
    <%= post.body %>
  </article>
<% end %>
```

The cache key is generated from the model's `cache_key` (includes `id` and `updated_at`). Updating the record automatically invalidates the cache.

Enable fragment caching in development:
```bash
rails dev:cache
```

## Russian Doll Caching

Nest cache blocks. The outer cache depends on the inner caches. When a comment changes, only that comment's fragment expires; the parent post fragment also expires because it touches `post.comments.maximum(:updated_at)`.

```erb
<%# Outer: expires when post or any of its comments change %>
<% cache [post, post.comments.maximum(:updated_at)] do %>
  <article>
    <h2><%= post.title %></h2>
    <% post.comments.each do |comment| %>
      <%# Inner: expires only when this comment changes %>
      <% cache comment do %>
        <p><%= comment.body %></p>
      <% end %>
    <% end %>
  </article>
<% end %>
```

## Low-Level Cache

Cache arbitrary objects — expensive queries, API calls, computed values.

```ruby
# Basic usage
result = Rails.cache.fetch('expensive_computation', expires_in: 1.hour) do
  ExpensiveService.compute
end

# Cache with a dynamic key
user_stats = Rails.cache.fetch("user/#{user.id}/stats/#{user.updated_at.to_i}") do
  UserStatsCalculator.new(user).call
end

# Delete a cache entry
Rails.cache.delete("user/#{user.id}/stats")

# Delete by pattern (Redis only)
Rails.cache.delete_matched("user/#{user.id}/*")
```

Cache key design rules:
- Include a version or timestamp when the underlying data can change
- Namespace by domain: `"posts/#{post.id}/summary"` not `"summary_#{post.id}"`
- Avoid cache keys that grow unboundedly (e.g., including full query params)

## HTTP Caching

Leverage the browser and CDN cache. Zero server load for repeat requests.

```ruby
# app/controllers/posts_controller.rb
def show
  @post = Post.find(params[:id])

  # Returns 304 Not Modified if the client has the current version
  if stale?(@post)
    respond_to do |format|
      format.html
      format.json { render json: @post }
    end
  end
end

# Manual ETag + Last-Modified
def show
  @post = Post.find(params[:id])
  fresh_when(etag: @post, last_modified: @post.updated_at, public: true)
end
```

For public pages (no user-specific content):
```ruby
expires_in 10.minutes, public: true
```

## Redis Configuration

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url:              ENV['REDIS_URL'],
  expires_in:       1.hour,
  namespace:        'cache',
  error_handler:    ->(method:, returning:, exception:) {
    Rails.logger.error "Redis cache error: #{exception.message}"
  }
}
```

## Counter and Rate Caches

```ruby
# Atomic increment — safe for concurrent requests
Rails.cache.increment("page_views/#{page.id}")
Rails.cache.decrement("active_users")

# Read the value
Rails.cache.read("page_views/#{page.id}")
```

## Cache Invalidation

The hardest part. Prefer automatic invalidation over manual:

```ruby
# Good: key includes updated_at — invalidates automatically on save
Rails.cache.fetch("post/#{post.id}/#{post.updated_at.to_i}") { expensive }

# Acceptable: explicit delete in a callback
class Post < ApplicationRecord
  after_update :clear_cache
  private
  def clear_cache
    Rails.cache.delete("post/#{id}/summary")
  end
end

# Risky: sweepers, observers, manual delete calls spread across the codebase
# These get out of sync. Use only when automatic invalidation is not possible.
```
