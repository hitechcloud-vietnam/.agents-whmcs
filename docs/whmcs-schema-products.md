# WHMCS Products Database Schema

Complete reference for the products table and related structures in WHMCS.

## Overview

The products tables manage all products, pricing, configuration options, and groups.

## Main Tables

### tblproducts

Product definitions.

```sql
CREATE TABLE `tblproducts` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `type` VARCHAR(50) NOT NULL DEFAULT 'hosting',
  `gid` INT(10) UNSIGNED DEFAULT NULL,
  `name` VARCHAR(255) NOT NULL,
  `slug` VARCHAR(255) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `descriptionHtml` TEXT DEFAULT NULL,
  `module` VARCHAR(50) DEFAULT NULL,
  `servergroup` INT(10) UNSIGNED DEFAULT NULL,
  `hidden` TINYINT(1) NOT NULL DEFAULT 0,
  `showdomainoptions` TINYINT(1) NOT NULL DEFAULT 0,
  `stockcontrol` TINYINT(1) NOT NULL DEFAULT 0,
  `stocklevel` INT(10) NOT NULL DEFAULT 0,
  `is_featured` TINYINT(1) NOT NULL DEFAULT 0,
  `welcome_email_id` INT(10) UNSIGNED DEFAULT NULL,
  `order` INT(10) UNSIGNED DEFAULT NULL,
  `retired` TINYINT(1) NOT NULL DEFAULT 0,
  `pricing_edit_mode` VARCHAR(50) DEFAULT 'inherit',
  `configoption_edit_mode` VARCHAR(50) DEFAULT 'inherit',
  `default_quantities` TINYINT(1) DEFAULT 0,
  `min_quantity` INT(10) DEFAULT NULL,
  `max_quantity` INT(10) DEFAULT NULL,
  `billing_cycle_setup_fee` VARCHAR(50) DEFAULT 'inherit',
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_gid` (`gid`),
  KEY `idx_type` (`type`),
  KEY `idx_hidden` (`hidden`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Product Types:**
| Type | Description |
|------|-------------|
| hosting | Standard hosting account |
| resellerhosting | Reseller hosting account |
| vps | Virtual private server |
| server | Dedicated server |
| other | Other product type |
| domain | Domain registration |

### tblproductgroups

Product groups/categories.

```sql
CREATE TABLE `tblproductgroups` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `headline` VARCHAR(255) DEFAULT NULL,
  `tagline` TEXT DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `hidden` TINYINT(1) NOT NULL DEFAULT 0,
  `order` INT(10) UNSIGNED DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblpricing

Product pricing by currency.

```sql
CREATE TABLE `tblpricing` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `relid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `type` VARCHAR(50) NOT NULL,
  `currency` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `msetup` DECIMAL(10,2) DEFAULT NULL,
  `qsetup` DECIMAL(10,2) DEFAULT NULL,
  `ssetup` DECIMAL(10,2) DEFAULT NULL,
  `asetup` DECIMAL(10,2) DEFAULT NULL,
  `bsetup` DECIMAL(10,2) DEFAULT NULL,
  `tsetup` DECIMAL(10,2) DEFAULT NULL,
  `monthly` DECIMAL(10,2) DEFAULT NULL,
  `quarterly` DECIMAL(10,2) DEFAULT NULL,
  `semiannually` DECIMAL(10,2) DEFAULT NULL,
  `annually` DECIMAL(10,2) DEFAULT NULL,
  `biennially` DECIMAL(10,2) DEFAULT NULL,
  `triennially` DECIMAL(10,2) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_relid_type` (`relid`, `type`),
  KEY `idx_currency` (`currency`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Pricing Types:**
- `product` - Product pricing
- `addon` - Addon pricing
- `domainregister` - Domain registration pricing
- `domainrenew` - Domain renewal pricing
- `domaintransfer` - Domain transfer pricing
- `configoption` - Configurable option pricing

### tblproductconfigoptions

Configuration option groups.

```sql
CREATE TABLE `tblproductconfigoptions` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `gid` INT(10) UNSIGNED DEFAULT NULL,
  `productid` INT(10) UNSIGNED DEFAULT NULL,
  `optionname` VARCHAR(255) NOT NULL,
  `optiontype` INT(10) NOT NULL DEFAULT 1,
  `qty_minimum` INT(10) DEFAULT NULL,
  `qty_maximum` INT(10) DEFAULT NULL,
  `qty_minimum_default` INT(10) DEFAULT NULL,
  `qty_maximum_default` INT(10) DEFAULT NULL,
  `order` INT(10) UNSIGNED DEFAULT NULL,
  `hidden` TINYINT(1) DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_gid` (`gid`),
  KEY `idx_productid` (`productid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Option Types:**
| Type | Description |
|------|-------------|
| 1 | Dropdown |
| 2 | Checkbox |
| 3 | Radio |
| 4 | Text |
| 5 | Textarea |
| 6 | Password |

### tblproductconfigoptions_sub

Configuration option values.

```sql
CREATE TABLE `tblproductconfigoptions_sub` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `configid` INT(10) UNSIGNED NOT NULL,
  `optionname` VARCHAR(255) NOT NULL,
  `sortorder` INT(10) DEFAULT NULL,
  `hidden` TINYINT(1) DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_configid` (`configid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblproductconfiglinks

Product to config option links.

```sql
CREATE TABLE `tblproductconfiglinks` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `gid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `productid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `configid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_productid` (`productid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbladdons

Product addons.

```sql
CREATE TABLE `tbladdons` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `module` VARCHAR(50) DEFAULT NULL,
  `access_group` VARCHAR(50) DEFAULT NULL,
  `name` VARCHAR(255) NOT NULL,
  `description` TEXT DEFAULT NULL,
  `billing_cycle` VARCHAR(50) DEFAULT NULL,
  `pricing` TEXT DEFAULT NULL,
  `welcome_email_template_id` INT(10) UNSIGNED DEFAULT NULL,
  `download_id` INT(10) UNSIGNED DEFAULT NULL,
  `hidden` TINYINT(1) NOT NULL DEFAULT 0,
  `show_order` TINYINT(1) NOT NULL DEFAULT 1,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblcustomfields

Custom fields for products.

```sql
CREATE TABLE `tblcustomfields` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `relid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `type` VARCHAR(50) NOT NULL DEFAULT 'product',
  `fieldname` VARCHAR(255) NOT NULL,
  `fieldtype` VARCHAR(50) NOT NULL DEFAULT 'text',
  `description` TEXT DEFAULT NULL,
  `fieldoptions` TEXT DEFAULT NULL,
  `regexpr` VARCHAR(255) DEFAULT NULL,
  `required` TINYINT(1) NOT NULL DEFAULT 0,
  `showorder` TINYINT(1) NOT NULL DEFAULT 1,
  `adminonly` TINYINT(1) NOT NULL DEFAULT 0,
  `sortorder` INT(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_relid_type` (`relid`, `type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblcustomfieldsvalues

Custom field values.

```sql
CREATE TABLE `tblcustomfieldsvalues` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `fieldid` INT(10) UNSIGNED NOT NULL,
  `relid` INT(10) UNSIGNED NOT NULL,
  `value` TEXT DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_fieldid_relid` (`fieldid`, `relid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get product with pricing

```php
function getProductWithPricing(int $productId, int $currencyId = 1): array
{
    $product = Capsule::table('tblproducts')
        ->where('id', $productId)
        ->first();
    
    $pricing = Capsule::table('tblpricing')
        ->where('relid', $productId)
        ->where('type', 'product')
        ->where('currency', $currencyId)
        ->first();
    
    return [
        'product' => $product,
        'pricing' => (array) $pricing
    ];
}
```

### Get products in group

```php
function getProductsInGroup(int $groupId): array
{
    return Capsule::table('tblproducts')
        ->where('gid', $groupId)
        ->where('hidden', 0)
        ->orderBy('order', 'asc')
        ->get();
}
```

### Get config options for product

```php
function getProductConfigOptions(int $productId): array
{
    return Capsule::table('tblproductconfigoptions')
        ->select('tblproductconfigoptions.*', 'tblproductconfiggroups.name as group_name')
        ->leftJoin('tblproductconfiggroups', 'tblproductconfiggroups.id', '=', 'tblproductconfigoptions.gid')
        ->where('tblproductconfigoptions.productid', $productId)
        ->orderBy('tblproductconfiggroups.order', 'asc')
        ->get();
}
```

## Related Documentation

- [whmcs-functions-products.md](../functions/whmcs-functions-products.md)
- [whmcs-module-provisioning-api.md](../modules/whmcs-module-provisioning-api.md)