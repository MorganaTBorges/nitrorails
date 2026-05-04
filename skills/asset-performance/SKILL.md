---
name: asset-performance
description: Use when optimizing Rails asset delivery — bundle size analysis, lazy loading, CDN configuration, compression (gzip/brotli), image optimization, HTTP/2, asset caching headers.
---

## Bundle Size Analysis

Before optimizing, measure what you're serving.

### With Propshaft or Sprockets

```bash
# List all assets and their sizes
rails assets:precompile
ls -lh public/assets/ | sort -k5 -hr | head -20
```

### With importmap (Rails 7+)

```bash
# Check what's imported
cat config/importmap.rb
# Identify large packages
```

### With esbuild / Vite

```bash
# esbuild bundle analysis
node_modules/.bin/esbuild app/javascript/application.js \
  --bundle --analyze --outfile=/dev/null 2>&1 | head -40
```

Red flags: any single JS file > 200KB (gzipped), any CSS file > 50KB.

## Lazy Loading JavaScript

```javascript
// Bad: imports the entire library at boot
import Chart from 'chart.js'

// Good: import only when needed
async function renderChart(data) {
  const { Chart } = await import('chart.js')
  new Chart(element, { data })
}
```

With importmap:
```html
<%# Only load heavy libraries on the pages that need them %>
<% content_for :head do %>
  <%= javascript_import_module_tag "charts" %>
<% end %>
```

## Image Optimization

```ruby
# In views — serve appropriately sized images
<%= image_tag post.cover.variant(resize_to_limit: [800, 600]), loading: "lazy" %>

# For responsive images
<%= image_tag post.cover.variant(resize_to_limit: [400, 300]),
    srcset: "#{url_for(post.cover.variant(resize_to_limit: [800, 600]))} 2x",
    loading: "lazy",
    width: 400, height: 300 %>
```

Always set explicit `width` and `height` to prevent layout shift (CLS).

## CDN Configuration

```ruby
# config/environments/production.rb
config.asset_host = ENV['CDN_HOST']  # e.g., 'https://cdn.yourapp.com'
```

With Cloudfront, set these cache behaviors:
- TTL: 1 year for fingerprinted assets (`/assets/*`)
- TTL: 1 hour for non-fingerprinted assets (`/packs/*`)
- Compress objects automatically: enabled
- Allowed HTTP methods: GET, HEAD

Fingerprinted assets (Sprockets/Propshaft append a hash to the filename) are safe to cache forever — they never change at the same URL.

## HTTP Compression

```ruby
# config/environments/production.rb
# Rack::Deflater adds gzip compression
config.middleware.use Rack::Deflater

# Or in nginx/load balancer (preferred — no Ruby CPU cost)
# gzip on;
# gzip_types text/plain text/css application/json application/javascript;
```

Check that compression is working:
```bash
curl -I -H "Accept-Encoding: gzip" https://yourapp.com/assets/application.js
# Look for: Content-Encoding: gzip
```

Brotli compression (better ratio than gzip, supported by all modern browsers):
```ruby
# Gemfile
gem 'rack-brotli'

# config/environments/production.rb
config.middleware.insert_after Rack::Deflater, Rack::Brotli
```

## Asset Caching Headers

Sprockets and Propshaft fingerprint assets automatically. Verify the headers are correct:

```bash
curl -I https://yourapp.com/assets/application-abc123.css
# Should include:
# Cache-Control: public, max-age=31536000, immutable
# ETag: "..."
```

For non-fingerprinted assets:
```ruby
# app/controllers/application_controller.rb
before_action :set_cache_headers

private

def set_cache_headers
  expires_in 10.minutes, public: true if request.format.html?
end
```

## HTTP/2 Server Push (when using nginx/CDN)

With HTTP/2, the server can push CSS and JS before the browser asks for them.

```nginx
# nginx
http2_push_preload on;
```

```ruby
# In Rails controller or view
response.headers['Link'] = [
  "</assets/application.css>; rel=preload; as=style",
  "</assets/application.js>; rel=preload; as=script"
].join(', ')
```

## Measurement

Before and after every asset change, measure:

```bash
# Time to first byte + total page load
curl -w "@curl-format.txt" -o /dev/null -s https://yourapp.com/your-page

# curl-format.txt contents:
#     time_namelookup:  %{time_namelookup}s
#        time_connect:  %{time_connect}s
#     time_appconnect:  %{time_appconnect}s
#    time_pretransfer:  %{time_pretransfer}s
#       time_redirect:  %{time_redirect}s
#  time_starttransfer:  %{time_starttransfer}s
#          time_total:  %{time_total}s
```

Use WebPageTest or Lighthouse for full browser-level measurement including render-blocking resources.
