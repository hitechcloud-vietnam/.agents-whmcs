# WHMCS Gateways Database Schema

Complete reference for the payment gateway tables in WHMCS.

## Main Tables

### tblpaymentgateways

Payment gateway configuration.

```sql
CREATE TABLE `tblpaymentgateways` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `gateway` VARCHAR(50) NOT NULL,
  `setting` VARCHAR(100) NOT NULL,
  `value` TEXT DEFAULT NULL,
  `order` INT(10) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_gateway` (`gateway`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblgatewayapi

Gateway API log.

```sql
CREATE TABLE `tblgatewayapi` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `gateway` VARCHAR(50) NOT NULL,
  `date` DATETIME DEFAULT NULL,
  `data` TEXT DEFAULT NULL,
  `success` TINYINT(1) DEFAULT 0,
  `response` TEXT DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_gateway` (`gateway`),
  KEY `idx_date` (`date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbltransections

Transaction records.

```sql
CREATE TABLE `tbltransections` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `currency` INT(10) UNSIGNED DEFAULT 0,
  `gateway` VARCHAR(50) DEFAULT NULL,
  `date` DATETIME DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `amountin` DECIMAL(10,2) DEFAULT 0.00,
  `amountout` DECIMAL(10,2) DEFAULT 0.00,
  `rate` DECIMAL(10,5) DEFAULT 1.00000,
  `transid` VARCHAR(255) DEFAULT NULL,
  `invoiceid` INT(10) UNSIGNED DEFAULT NULL,
  `refundid` INT(10) UNSIGNED DEFAULT NULL,
  `fee` DECIMAL(10,2) DEFAULT 0.00,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`),
  KEY `idx_gateway` (`gateway`),
  KEY `idx_invoiceid` (`invoiceid`),
  KEY `idx_date` (`date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblcc信息

Credit card information (encrypted).

```sql
CREATE TABLE `tblccinfo` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `userid` INT(10) UNSIGNED NOT NULL,
  `gateway` VARCHAR(50) DEFAULT NULL,
  `cardtype` VARCHAR(50) DEFAULT NULL,
  `cardlastfour` VARCHAR(10) DEFAULT NULL,
  `cardnum` VARCHAR(255) DEFAULT NULL,
  `expdate` VARCHAR(10) DEFAULT NULL,
  `startdate` VARCHAR(10) DEFAULT NULL,
  `issuenumber` VARCHAR(50) DEFAULT NULL,
  `encvar` TEXT DEFAULT NULL,
  `default` TINYINT(1) DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get gateway config

```php
function getGatewayConfig(string $gateway): array
{
    return Capsule::table('tblpaymentgateways')
        ->where('gateway', $gateway)
        ->pluck('value', 'setting')
        ->toArray();
}
```

### Get transactions by gateway

```php
function getGatewayTransactions(string $gateway, int $limit = 100): array
{
    return Capsule::table('tbltransections')
        ->where('gateway', $gateway)
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
}
```

### Get client transactions

```php
function getClientTransactions(int $clientId): array
{
    return Capsule::table('tbltransections')
        ->where('userid', $clientId)
        ->orderBy('date', 'desc')
        ->get();
}
```

## Related Documentation

- [whmcs-module-gateway-api.md](../modules/whmcs-module-gateway-api.md)
- [whmcs-functions-transactions.md](../functions/whmcs-functions-transactions.md)