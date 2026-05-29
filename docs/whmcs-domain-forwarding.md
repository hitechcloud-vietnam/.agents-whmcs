# WHMCS Domain Forwarding

## Overview

Domain forwarding in WHMCS redirects visitors from one domain to another URL, useful for marketing domains or short URLs.

## Forwarding Configuration

### Enable Forwarding

**Configuration > Domains > Forwarding**

```php
// Forwarding settings
[
    'enable_forwarding' => false,
    'default_type' => '301',          // 301, 302
    'default_target' => 'https://...',
    'meta_refresh_seconds' => 0
]
```

## Forward Types

### HTTP Redirects

| Type | Description | Use Case |
|------|-------------|----------|
| 301 | Permanent redirect | SEO, old domain |
| 302 | Temporary redirect | Seasonal redirects |
| Meta Refresh | HTML refresh | When redirect fails |

## Forward Setup

### Configure Forward

```php
// Domain forward
[
    'domain_id' => 1,
    'source_domain' => 'short.com',
    'target_url' => 'https://longdomain.com/page',
    'redirect_type' => '301',
    'forward_emails' => true,
    'email_forward_to' => 'info@example.com'
]
```

## DNS Configuration

### CNAME Setup

```php
// DNS for forwarding
[
    'domain' => 'short.com',
    'cname' => 'redirect.yourcompany.com'
]
```

## Forward Options

### Additional Options

```php
// Forwarding options
[
    'path_forwarding' => true,          // Include path
    'query_forwarding' => true,         // Include query string
    'forward_emails' => true,
    'email_target' => 'sales@example.com',
    'cloaking' => false
]
```

## API Functions

```php
// Set domain forwarding
$result = localAPI('SetDomainForwarding', [
    'domainid' => 1,
    'target_url' => 'https://example.com',
    'type' => '301'
]);

// Remove forwarding
$result = localAPI('RemoveDomainForwarding', [
    'domainid' => 1
]);
```

## Best Practices

1. **Use 301 for permanent**: Better for SEO
2. **Monitor forwards**: Track redirect performance
3. **Forward emails**: Don't lose email
4. **Keep updated**: Update when targets change

## Related Documentation

- [Domain Nameservers](./whmcs-domain-nameservers.md)
- [Domain Park Page](./whmcs-domain-park-page.md)
- [Domain Registration](./whmcs-domain-registration.md)