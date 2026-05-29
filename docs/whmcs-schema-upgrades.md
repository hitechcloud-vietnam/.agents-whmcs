# WHMCS Upgrades Schema

Complete reference for service upgrade/downgrade tables in WHMCS.

## Main Tables

### tblupgrades

Upgrade records.

```sql
CREATE TABLE `tblupgrades` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `orderid` INT(10) UNSIGNED DEFAULT NULL,
  `entityid` INT(10) UNSIGNED DEFAULT NULL,
  `entitytype` VARCHAR(50) DEFAULT 'service',
  `original_value` DECIMAL(10,2) DEFAULT NULL,
  `new_value` DECIMAL(10,2) DEFAULT NULL,
  `recurring_change` DECIMAL(10,2) DEFAULT NULL,
  `setup_fee` DECIMAL(10,2) DEFAULT NULL,
  `billing_cycle` VARCHAR(50) DEFAULT NULL,
  `new_billing_cycle` VARCHAR(50) DEFAULT NULL,
  `type` VARCHAR(50) DEFAULT 'upgrade',
  `status` VARCHAR(50) DEFAULT 'Pending',
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_entity` (`entityid`, `entitytype`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Upgrade Types

| Type | Description |
|------|-------------|
| upgrade | Product upgrade |
| downgrade | Product downgrade |
| configoptions | Config option changes |
| billingcycle | Billing cycle change |

### tblservice_upgrades

Service-specific upgrades.

```sql
CREATE TABLE `tblservice_upgrades` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `hosting_id` INT(10) UNSIGNED NOT NULL,
  `order_id` INT(10) UNSIGNED NOT NULL,
  `original_product_id` INT(10) UNSIGNED DEFAULT NULL,
  `new_product_id` INT(10) UNSIGNED DEFAULT NULL,
  `type` VARCHAR(50) DEFAULT 'upgrade',
  `amount` DECIMAL(10,2) DEFAULT 0.00,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_hosting_id` (`hosting_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get pending upgrades

```php
function getPendingUpgrades(): array
{
    return Capsule::table('tblupgrades')
        ->where('status', 'Pending')
        ->get();
}
```

### Calculate upgrade price

```php
function calculateUpgradePrice(
    int $currentProductId,
    int $newProductId,
    string $billingCycle,
    int $currencyId = 1
): array {
    $currentPricing = Capsule::table('tblpricing')
        ->where('relid', $currentProductId)
        ->where('type', 'product')
        ->where('currency', $currencyId)
        ->first();
    
    $newPricing = Capsule::table('tblpricing')
        ->where('relid', $newProductId)
        ->where('type', 'product')
        ->where('currency', $currencyId)
        ->first();
    
    $cycleColumn = strtolower($billingCycle);
    $currentPrice = $currentPricing->$cycleColumn ?? 0;
    $newPrice = $newPricing->$cycleColumn ?? 0;
    
    $priceDifference = $newPrice - $currentPrice;
    
    return [
        'current_price' => $currentPrice,
        'new_price' => $newPrice,
        'difference' => $priceDifference,
        'prorated' => $priceDifference / 2 // Example prorate calculation
    ];
}
```

## Related Documentation

- [whmcs-schema-services.md](whmcs-schema-services.md)
- [whmcs-functions-orders.md](../functions/whmcs-functions-orders.md)