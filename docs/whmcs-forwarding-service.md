# WHMCS Domain Forwarding Service Documentation

## Overview

Domain forwarding allows you to redirect visitors from one domain to another URL. This documentation covers setup, configuration, and management of domain forwarding in WHMCS.

## Forwarding Types

### 301 Permanent Redirect

- Best for SEO preservation
- Permanent redirect signal to search engines
- Transfers most link equity
- Recommended for moved permanently content

### 302 Temporary Redirect

- Temporary redirect
- Search engines may still index original
- Use for temporary promotions
- A/B testing scenarios

### 303 See Other

- Used for POST-then-redirect pattern
- Prevents form re-submission on refresh
- Common in e-commerce checkouts

### Meta Refresh

- Client-side redirect via HTML meta tag
- Works when JavaScript is disabled
- Slower than 301/302
- Limited SEO value

## Configuration

### Enable Forwarding

Navigate to: **Setup > Products/Services > Domain Pricing**

Enable forwarding for specific TLDs:

| TLD | Forwarding Available | Default Type |
|-----|---------------------|--------------|
| .com | Yes | 301 |
| .net | Yes | 301 |
| .org | Yes | 301 |
| .info | Yes | 302 |
| .biz | Yes | 301 |

### Global Settings

**Configuration > General Settings > Domains**

```php
// Forwarding Configuration
$forwardingConfig = [
    'enabled' => true,
    'default_type' => 301,
    'allow_permanent' => true,
    'allow_temporary' => true,
    'allow_meta_refresh' => true,
    'max_redirects_per_domain' => 5,
    'forwarding_timeout' => 5,          // seconds
    'preserve_path' => true,             // Forward /old/page to /new/page
    'preserve_query' => true,            // Forward query strings
    'mask_destination' => false,         // Hide destination URL
    'nofollow_outbound' => false,        // Add nofollow to masked
    'cloaking_enabled' => false         // Show content instead of redirect
];
```

## Customer Portal

### Add Forwarding

**Client Area > My Domains > Manage > URL Forwarding**

```
+------------------------------------------+
| Add URL Forwarding                       |
+------------------------------------------+
| Source Subdomain: [www______________]   |
| Destination URL: [https://newsite.com__]|
| Forwarding Type: [301 Permanent____]    |
|                                          |
| Options:                                 |
| [x] Forward path (preserve URL path)    |
| [x] Forward query string                 |
| [ ] Mask destination URL                 |
|                                          |
|                          [Add Forward]  |
+------------------------------------------+
```

### Forwarding Options

| Option | Description | Use Case |
|--------|-------------|----------|
| Forward Path | Append source path to destination | /blog/* to new site |
| Forward Query | Include URL parameters | Track marketing sources |
| Mask URL | Show source URL in browser | Branded forwarding |
| NoFollow | Add rel="nofollow" | Prevent link equity transfer |

### Masked Forwarding

When URL masking is enabled:

```
+------------------------------------------+
| Masked Forwarding Settings               |
+------------------------------------------+
| Title: [My Business Website__________]   |
| Meta Description:                        |
| [____________________________________]   |
| Default Content:                         |
| [____________________________________]   |
| [x] Use iframe for content              |
| [ ] Enable JavaScript redirect fallback  |
|                                          |
|                          [Save Settings]  |
+------------------------------------------+
```

## API Integration

### Create Forwarding Rule

```http
POST /domains/{domain}/forwarding
```

**Request Body:**

```json
{
  "source": {
    "subdomain": "www",
    "path": "/blog"
  },
  "destination": "https://newsite.com/posts",
  "type": "301",
  "options": {
    "preserve_path": true,
    "preserve_query": true,
    "mask": false
  }
}
```

**Response:**

```json
{
  "success": true,
  "forwarding_id": "FWD-12345",
  "source": "www.example.com/blog",
  "destination": "https://newsite.com/posts/blog",
  "type": "301",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### List Forwarding Rules

```http
GET /domains/{domain}/forwarding
```

**Response:**

```json
{
  "domain": "example.com",
  "forwarding_rules": [
    {
      "id": "FWD-12345",
      "source": "www.example.com",
      "destination": "https://newsite.com",
      "type": "301",
      "created_at": "2024-01-15T10:30:00Z"
    },
    {
      "id": "FWD-12346",
      "source": "blog.example.com",
      "destination": "https://newsite.com/blog",
      "type": "302",
      "created_at": "2024-01-16T14:00:00Z"
    }
  ]
}
```

### Update Forwarding Rule

```http
PUT /domains/{domain}/forwarding/{forwarding_id}
```

```json
{
  "destination": "https://newsite.com/updated",
  "type": "301",
  "options": {
    "preserve_path": true,
    "preserve_query": false
  }
}
```

### Delete Forwarding Rule

```http
DELETE /domains/{domain}/forwarding/{forwarding_id}
```

## Implementation Methods

### Server-Side (Recommended)

#### Apache .htaccess

```apache
# 301 Permanent Redirect
RewriteEngine On
RewriteCond %{HTTP_HOST} ^www\.example\.com$ [NC]
RewriteRule ^(.*)$ https://newsite.com/$1 [R=301,L]

# 302 Temporary Redirect
RewriteEngine On
RewriteCond %{HTTP_HOST} ^blog\.example\.com$ [NC]
RewriteRule ^(.*)$ https://newsite.com/posts/$1 [R=302,L]
```

#### Nginx Configuration

```nginx
# 301 Permanent Redirect
server {
    server_name www.example.com;
    return 301 https://newsite.com$request_uri;
}

# 302 Temporary Redirect
server {
    server_name blog.example.com;
    return 302 https://newsite.com/posts$request_uri;
}
```

### PHP Implementation

```php
// 301 Permanent Redirect
function permanentRedirect(string $url, bool $preservePath = true) {
    if ($preservePath && isset($_SERVER['REQUEST_URI'])) {
        $url .= $_SERVER['REQUEST_URI'];
    }

    header('HTTP/1.1 301 Moved Permanently');
    header('Location: ' . $url);
    header('Cache-Control: no-store, no-cache, must-revalidate');
    exit;
}

// 302 Temporary Redirect
function temporaryRedirect(string $url, bool $preservePath = true) {
    if ($preservePath && isset($_SERVER['REQUEST_URI'])) {
        $url .= $_SERVER['REQUEST_URI'];
    }

    header('HTTP/1.1 302 Found');
    header('Location: ' . $url);
    exit;
}
```

### Masked Forwarding (iframe)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Redirecting...</title>
    <meta name="description" content="Your page description">
    <style>
        body { margin: 0; padding: 0; height: 100vh; }
        iframe { width: 100%; height: 100%; border: none; }
    </style>
</head>
<body>
    <iframe src="https://destination.com" sandbox="allow-scripts allow-same-origin">
        <p>Your browser doesn't support iframes.
        <a href="https://destination.com">Click here to continue</a></p>
    </iframe>
</body>
</html>
```

## SEO Considerations

### Preserving SEO Value

| Action | Impact |
|--------|--------|
| Use 301 redirect | Preserves ~90% link equity |
| Use 302 redirect | No equity transfer |
| Update internal links | Best long-term solution |
| Update XML sitemap | Tell search engines new location |
| Use canonical tags | Prevent duplicate content |

### Best Practices

1. **Use 301 for permanent moves**
   - Content moved permanently
   - Domain name change
   - Website restructure

2. **Use 302 for temporary**
   - Seasonal promotions
   - A/B testing
   - Maintenance pages

3. **Preserve URL structure when possible**
   - `/blog/post` -> `/articles/post`
   - Redirect old paths individually

4. **Update after migration**
   - Update internal links
   - Submit new sitemap
   - Monitor search console

## Wildcard Forwarding

### Wildcard Forwarding Rules

```php
// Wildcard forwarding configuration
$wildcardConfig = [
    'enabled' => true,
    'pattern' => '*.example.com',
    'destination' => 'https://newsite.com/$1',
    'type' => '301',
    'strip_subdomain' => false
];
```

**Examples:**
- `blog.example.com/*` -> `https://newsite.com/blog/*`
- `shop.example.com/*` -> `https://newsite.com/store/*`
- `*.example.com` -> `https://newsite.com/sites/$1`

## Batch Forwarding

### Bulk Create Forwarding

```http
POST /domains/{domain}/forwarding/batch
```

**Request Body:**

```json
{
  "rules": [
    {
      "source": "www",
      "destination": "https://newsite.com",
      "type": "301"
    },
    {
      "source": "blog",
      "destination": "https://blog.newhost.com",
      "type": "301"
    },
    {
      "source": "shop",
      "destination": "https://store.newhost.com",
      "type": "301"
    }
  ]
}
```

### Domain Migration Tool

```php
// Complete domain migration with forwarding
$migration = WHMCS\Domains\Forwarding\Migration::create([
    'source_domain' => 'oldsite.com',
    'destination_domain' => 'newsite.com',
    'redirect_type' => '301',
    'include_subdomains' => true,
    'preserve_structure' => true,
    'duration' => 90  // days to keep forwarding
]);
```

## Monitoring

### Forwarding Statistics

**Admin Dashboard > Domain Forwarding > Statistics**

| Metric | Description |
|--------|-------------|
| Total Forwards | Active forwarding rules |
| Hits Today | Forwarded requests today |
| Popular Destinations | Most common destinations |
| Error Rate | Failed redirects |

### Click Tracking

```php
// Enable click tracking
$tracking = WHMCS\Domains\Forwarding\Tracker::enable('FWD-12345');

// Get statistics
$stats = WHMCS\Domains\Forwarding\Tracker::getStats('FWD-12345', [
    'period' => 'last_30_days'
]);

// Response
[
    'total_clicks' => 15420,
    'unique_clicks' => 8934,
    'top_sources' => [
        'google.com' => 5420,
        'facebook.com' => 3210,
        'twitter.com' => 1890
    ],
    'time_series' => [
        ['date' => '2024-01-01', 'clicks' => 520],
        ['date' => '2024-01-02', 'clicks' => 480]
    ]
]
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Redirect loop | Source equals destination | Check destination URL |
| 404 on destination | Wrong path mapping | Verify destination exists |
| Masking broken | iframe blocked | Provide fallback link |
| SEO drop | Using 302 instead of 301 | Change to 301 |
| Slow redirect | DNS propagation | Lower TTL before changes |

### Debug Mode

```bash
# Check forwarding status
whmcscli forwarding status --domain=example.com

# Test redirect
whmcscli forwarding test --source=www.example.com/test

# View forwarding logs
whmcscli forwarding logs --domain=example.com --limit=50
```

### Validation

```php
// Validate forwarding configuration
$validation = WHMCS\Domains\Forwarding\Validator::check([
    'source' => 'www.example.com',
    'destination' => 'https://newsite.com',
    'type' => '301'
]);

if (!$validation['valid']) {
    foreach ($validation['errors'] as $error) {
        echo "Error: $error\n";
    }
}
```

## Limits and Pricing

### Service Limits

| Plan | Max Forwarding Rules | Wildcard Allowed |
|------|---------------------|------------------|
| Basic | 5 | No |
| Standard | 20 | Yes |
| Premium | 100 | Yes |
| Enterprise | Unlimited | Yes |

### Pricing Options

| Model | Price | Description |
|-------|-------|-------------|
| Included | Free | Included with domain |
| Per Rule | $0.50/rule/month | Additional rules |
| Bulk Package | $9.99/50 rules/month | Volume pricing |
| Unlimited | $19.99/month | Unlimited forwarding |

## See Also

- [Park Page Setup](./whmcs-park-page.md)
- [DNS Management API](./whmcs-dns-management-api.md)
- [Domain Pricing Configuration](../products-services/domain-pricing.md)
