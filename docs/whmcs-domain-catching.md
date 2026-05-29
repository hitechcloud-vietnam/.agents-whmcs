# WHMCS Domain Catching

## Overview

Domain catching (drop catching) in WHMCS monitors expiring domains and attempts to register them the moment they become available after expiration.

## Drop Catching Configuration

### Enable Catching

**Configuration > Domains > Drop Catching**

```php
// Drop catch settings
[
    'enable' => false,
    'api_enabled' => false,
    'providers' => [
        'dropcatch' => ['enabled' => false],
        'namejet' => ['enabled' => false]
    ]
]
```

## Catch Process

### Monitor and Catch

```php
// Drop catch process
[
    'monitor_tlds' => ['com', 'net', 'org'],
    'catch_attempts' => 3,
    'max_bid' => 100.00,
    'use_premium' => false
]
```

## Catch Configuration

### Provider Settings

```php
// Drop catch provider
[
    'provider' => 'namejet',
    'api_key' => 'xxx',
    'auto_catch' => true,
    'notification' => true
]
```

## API Functions

```php
// Configure drop catch
$result = localAPI('ConfigureDropCatch', [
    'enabled' => true,
    'provider' => 'namejet'
]);
```

## Best Practices

1. **Use providers**: Leverage drop catch services
2. **Monitor TLDs**: Focus on valuable TLDs
3. **Set limits**: Define maximum bid prices
4. **Fast action**: Register quickly when available

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Auction](./whmcs-domain-auction.md)
- [Domain Backorder](./whmcs-domain-backorder.md)