# WHMCS Quotes Database Schema

Complete reference for quotes tables in WHMCS.

## Main Tables

### tblquotes

Quote header information.

```sql
CREATE TABLE `tblquotes` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `subject` VARCHAR(255) NOT NULL,
  `stage` VARCHAR(50) NOT NULL DEFAULT 'Draft',
  `number` VARCHAR(50) DEFAULT NULL,
  `date_created` DATE DEFAULT NULL,
  `date_expires` DATE DEFAULT NULL,
  `date_accepted` DATE DEFAULT NULL,
  `valid_tdays` INT(10) DEFAULT NULL,
  `subtotal` DECIMAL(10,2) DEFAULT 0.00,
  `taxrate` DECIMAL(5,2) DEFAULT NULL,
  `taxrate2` DECIMAL(5,2) DEFAULT NULL,
  `tax` DECIMAL(10,2) DEFAULT 0.00,
  `total` DECIMAL(10,2) DEFAULT 0.00,
  `currency` INT(10) UNSIGNED DEFAULT 1,
  `client_id` INT(10) UNSIGNED DEFAULT NULL,
  `user_id` INT(10) UNSIGNED DEFAULT NULL,
  `firstname` VARCHAR(100) DEFAULT NULL,
  `lastname` VARCHAR(100) DEFAULT NULL,
  `companyname` VARCHAR(255) DEFAULT NULL,
  `email` VARCHAR(255) DEFAULT NULL,
  `address1` VARCHAR(255) DEFAULT NULL,
  `address2` VARCHAR(255) DEFAULT NULL,
  `city` VARCHAR(100) DEFAULT NULL,
  `state` VARCHAR(100) DEFAULT NULL,
  `postcode` VARCHAR(20) DEFAULT NULL,
  `country` VARCHAR(2) DEFAULT NULL,
  `phonenumber` VARCHAR(30) DEFAULT NULL,
  `paymentmethod` VARCHAR(50) DEFAULT NULL,
  `notes` TEXT DEFAULT NULL,
  `admin_id` INT(10) UNSIGNED DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`),
  KEY `idx_stage` (`stage`),
  KEY `idx_date_expires` (`date_expires`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Quote Stages

| Stage | Description |
|-------|-------------|
| Draft | Quote in draft state |
| Delivered | Sent to client |
| On Hold | Placed on hold |
| Accepted | Accepted by client |
| Lost | Lost opportunity |
| Dead | No longer active |

### tblquoteitems

Quote line items.

```sql
CREATE TABLE `tblquoteitems` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `quote_id` INT(10) UNSIGNED NOT NULL,
  `description` TEXT DEFAULT NULL,
  `quantity` DECIMAL(10,2) DEFAULT 1.00,
  `unit_price` DECIMAL(10,2) DEFAULT 0.00,
  `discount` DECIMAL(10,2) DEFAULT 0.00,
  `taxable` TINYINT(1) DEFAULT 1,
  `order` INT(10) DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_quote_id` (`quote_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblquotes_lineitemtax

Quote item tax rates.

```sql
CREATE TABLE `tblquotes_lineitemtax` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `quote_item_id` INT(10) UNSIGNED NOT NULL,
  `name` VARCHAR(100) NOT NULL,
  `rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  PRIMARY KEY (`id`),
  KEY `idx_quote_item_id` (`quote_item_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get quote with items

```php
function getQuoteWithItems(int $quoteId): array
{
    $quote = Capsule::table('tblquotes')
        ->where('id', $quoteId)
        ->first();
    
    $items = Capsule::table('tblquoteitems')
        ->where('quote_id', $quoteId)
        ->orderBy('order', 'asc')
        ->get();
    
    return [
        'quote' => $quote,
        'items' => $items
    ];
}
```

### Get expired quotes

```php
function getExpiredQuotes(): array
{
    return Capsule::table('tblquotes')
        ->where('stage', '!=', 'Accepted')
        ->where('stage', '!=', 'Lost')
        ->where('stage', '!=', 'Dead')
        ->where('date_expires', '<', date('Y-m-d'))
        ->get();
}
```

## Related Documentation

- [whmcs-functions-utility.md](../functions/whmcs-functions-utility.md)