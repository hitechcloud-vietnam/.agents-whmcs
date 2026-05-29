# WHMCS TLD Sync

## Overview

TLD synchronization in WHMCS keeps the list of supported Top-Level Domains current with registry updates, pricing changes, and availability.

## TLD Sync Configuration

### Enable Sync

**Configuration > Domains > TLD Sync**

```php
// TLD sync settings
[
    'enable_sync' => true,
    'sync_frequency' => 'daily',
    'auto_update_pricing' => true,
    'auto_add_new' => true
]
```

## Sync Operations

### Sync TLD Data

```php
// Synchronize TLDs
[
    'sync_pricing' => true,
    'sync_availability' => true,
    'sync_requirements' => true,
    'sync_idn_support' => true
]
```

## TLD Updates

### Price Updates

```php
// Update TLD pricing
[
    'tld' => 'com',
    'old_pricing' => 9.95,
    'new_pricing' => 10.95,
    'effective_date' => '2024-06-01'
]
```

### New TLDs

```php
// Add new TLDs
[
    'tld' => 'xyz',
    'added_date' => '2024-05-15',
    'pricing' => 12.95,
    'id_protection' => true,
    'registration_periods' => [1, 2, 5, 10]
]
```

## Sync Cron

### Automated Sync

```bash
# TLD sync cron
0 3 * * * php -q /whmcs/crons/tldsync.php
```

## Registry Updates

### Stay Current

```php
// Registry changes
[
    'tld' => 'io',
    'registry' => 'dot.io',
    'requirements_changed' => false,
    'pricing_changed' => true
]
```

## API Functions

```php
// Sync TLDs
$result = localAPI('SyncTLDs');

// Get TLD updates
$result = localAPI('GetTLDUpdates', [
    'since' => '2024-05-01'
]);
```

## Best Practices

1. **Regular sync**: Keep TLDs current
2. **Monitor updates**: Track registry changes
3. **Update pricing**: Reflect registry costs
4. **Add new TLDs**: Expand offerings

## Related Documentation

- [Domain Sync](./whmcs-domain-sync.md)
- [Domain Pricing](./whmcs-domain-pricing.md)
- [Domain Registration](./whmcs-domain-registration.md)