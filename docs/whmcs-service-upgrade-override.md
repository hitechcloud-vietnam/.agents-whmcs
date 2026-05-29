# WHMCS Service Upgrade Override

## Overview

Service upgrade override in WHMCS allows administrators to set custom upgrade paths or prevent certain upgrades from being available.

## Override Configuration

### Disable Upgrades

**Admin: Clients > Services > Upgrade Settings**

```php
// Override upgrade
[
    'service_id' => 1,
    'upgrade_override' => true,
    'allowed_upgrades' => [2, 3],       // Only allow to products 2 and 3
    'disallowed_upgrades' => [],
    'reason' => 'Limited to specific tiers'
]
```

### Override Types

| Type | Description |
|------|-------------|
| allow_specific | Only specific upgrades allowed |
| disallow_specific | Block specific upgrades |
| complete_block | Block all upgrades |

## Allowed Upgrades

### Specific Products Only

```php
// Allow only certain upgrades
[
    'service_id' => 1,
    'override_type' => 'allow_specific',
    'allowed_products' => [2, 3],
    'reason' => 'Tiered upgrade path'
]
```

### Block Specific Products

```php
// Block certain upgrades
[
    'service_id' => 1,
    'override_type' => 'disallow_specific',
    'disallowed_products' => [10, 11],
    'reason' => 'Not compatible'
]
```

## Override Management

### Set Override

```php
// Add upgrade override
[
    'service_id' => 1,
    'type' => 'allow_specific',
    'allowed_upgrades' => [2, 3],
    'reason' => 'Premium tier only'
]
```

### Remove Override

```php
// Remove override
[
    'service_id' => 1,
    'action' => 'remove_override',
    'restore_all_upgrades' => true
]
```

## Custom Upgrade Pricing

### Override Upgrade Price

```php
// Custom upgrade pricing
[
    'service_id' => 1,
    'upgrade_to_product_id' => 2,
    'override_prorate' => 10.00,        // Custom proration
    'reason' => 'Special loyalty pricing'
]
```

## API Functions

```php
// Set upgrade override
$result = localAPI('SetUpgradeOverride', [
    'serviceid' => 1,
    'type' => 'allow_specific',
    'allowed_upgrades' => [2, 3]
]);

// Remove override
$result = localAPI('RemoveUpgradeOverride', [
    'serviceid' => 1
]);
```

## Best Practices

1. **Clear paths**: Define logical upgrade paths
2. **Document reasons**: Record why override was set
3. **Consider compatibility**: Ensure upgrades work
4. **Test upgrades**: Verify override works correctly

## Related Documentation

- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Override](./whmcs-service-override.md)
- [Product Configuration](./whmcs-product-configuration.md)
- [Service Upgrade Request](./whmcs-service-upgrade-request.md)