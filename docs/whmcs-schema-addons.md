# WHMCS Addons Database Schema

Complete reference for addon module tables in WHMCS.

## Main Tables

### tbladdons

Addon definitions.

```sql
CREATE TABLE `tbladdons` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `module` VARCHAR(50) DEFAULT NULL,
  `name` VARCHAR(255) NOT NULL,
  `description` TEXT DEFAULT NULL,
  `billing_cycle` VARCHAR(50) DEFAULT NULL,
  `pricing` TEXT DEFAULT NULL,
  `welcome_email_template_id` INT(10) UNSIGNED DEFAULT NULL,
  `download_id` INT(10) UNSIGNED DEFAULT NULL,
  `hidden` TINYINT(1) NOT NULL DEFAULT 0,
  `show_order` TINYINT(1) NOT NULL DEFAULT 1,
  `weight` INT(10) DEFAULT 0,
  `product_id` INT(10) UNSIGNED DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblhosting_addons

Service addons.

```sql
CREATE TABLE `tblhosting_addons` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `hostingid` INT(10) UNSIGNED NOT NULL,
  `addonid` INT(10) UNSIGNED NOT NULL,
  `orderid` INT(10) UNSIGNED DEFAULT NULL,
  `name` VARCHAR(255) DEFAULT NULL,
  `setup_fee` DECIMAL(10,2) DEFAULT NULL,
  `recurring` DECIMAL(10,2) DEFAULT NULL,
  `billingcycle` VARCHAR(50) DEFAULT NULL,
  `nextduedate` DATE DEFAULT NULL,
  `nextinvoicedate` DATE DEFAULT NULL,
  `status` VARCHAR(50) DEFAULT 'Pending',
  `automatic` TINYINT(1) NOT NULL DEFAULT 0,
  `payment_method` VARCHAR(50) DEFAULT NULL,
  `notes` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_hostingid` (`hostingid`),
  KEY `idx_addonid` (`addonid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbladdon_pricing

Addon pricing by currency.

```sql
CREATE TABLE `tbladdon_pricing` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `addon_id` INT(10) UNSIGNED NOT NULL,
  `currency` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `monthly` DECIMAL(10,2) DEFAULT NULL,
  `quarterly` DECIMAL(10,2) DEFAULT NULL,
  `semiannually` DECIMAL(10,2) DEFAULT NULL,
  `annually` DECIMAL(10,2) DEFAULT NULL,
  `biennially` DECIMAL(10,2) DEFAULT NULL,
  `triennially` DECIMAL(10,2) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_addon_currency` (`addon_id`, `currency`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get addon with pricing

```php
function getAddonWithPricing(int $addonId, int $currencyId = 1): array
{
    $addon = Capsule::table('tbladdons')
        ->where('id', $addonId)
        ->first();
    
    $pricing = Capsule::table('tbladdon_pricing')
        ->where('addon_id', $addonId)
        ->where('currency', $currencyId)
        ->first();
    
    return [
        'addon' => $addon,
        'pricing' => (array) $pricing
    ];
}
```

### Get service addons

```php
function getServiceAddons(int $serviceId): array
{
    return Capsule::table('tblhosting_addons')
        ->select('tblhosting_addons.*', 'tbladdons.name as addon_name')
        ->join('tbladdons', 'tbladdons.id', '=', 'tblhosting_addons.addonid')
        ->where('tblhosting_addons.hostingid', $serviceId)
        ->get();
}
```

## Related Documentation

- [whmcs-module-addon-api.md](../modules/whmcs-module-addon-api.md)