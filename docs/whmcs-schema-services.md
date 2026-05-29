# WHMCS Services Database Schema

Complete reference for the services (hosting) table and related structures in WHMCS.

## Overview

The services tables manage all active hosting accounts and services.

## Main Tables

### tblhosting

Primary services table.

```sql
CREATE TABLE `tblhosting` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `uuid` CHAR(36) DEFAULT NULL,
  `userid` INT(10) UNSIGNED NOT NULL,
  `orderid` INT(10) UNSIGNED DEFAULT NULL,
  `packageid` INT(10) UNSIGNED NOT NULL,
  `server` INT(10) UNSIGNED DEFAULT NULL,
  `regdate` DATETIME NOT NULL,
  `domainstatus` VARCHAR(50) NOT NULL DEFAULT 'Pending',
  `suspendreason` VARCHAR(255) DEFAULT NULL,
  `subscriptionid` VARCHAR(255) DEFAULT NULL,
  `firstpaymentamount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `billingcycle` VARCHAR(50) NOT NULL DEFAULT 'Monthly',
  `nextduedate` DATE DEFAULT NULL,
  `nextinvoicedate` DATE DEFAULT NULL,
  `domain` VARCHAR(255) DEFAULT NULL,
  `username` VARCHAR(255) DEFAULT NULL,
  `password` VARCHAR(255) DEFAULT NULL,
  `password_encrypted` VARCHAR(255) DEFAULT NULL,
  `notes` TEXT DEFAULT NULL,
  `diskusage` BIGINT(20) DEFAULT 0,
  `disklimit` BIGINT(20) DEFAULT 0,
  `bwusage` BIGINT(20) DEFAULT 0,
  `bwlimit` BIGINT(20) DEFAULT 0,
  `lastupdate` DATETIME DEFAULT NULL,
  `termination_date` DATETIME DEFAULT NULL,
  `requested_username` VARCHAR(255) DEFAULT NULL,
  `module` VARCHAR(50) DEFAULT NULL,
  `moduleid` INT(10) UNSIGNED DEFAULT NULL,
  `priority` INT(10) DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `email_verified_at` DATETIME DEFAULT NULL,
  `override_auto_suspend` TINYINT(1) NOT NULL DEFAULT 0,
  `override_auto_terminate` TINYINT(1) NOT NULL DEFAULT 0,
  `subscription_id` VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`),
  KEY `idx_packageid` (`packageid`),
  KEY `idx_server` (`server`),
  KEY `idx_domainstatus` (`domainstatus`),
  KEY `idx_nextduedate` (`nextduedate`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Service Statuses:**
| Status | Description |
|--------|-------------|
| Pending | Awaiting provisioning |
| Active | Service active |
| Suspended | Service suspended |
| Terminated | Service terminated |
| Cancelled | Service cancelled |

### tblhosting_additional

Service additional info.

```sql
CREATE TABLE `tblhosting_additional` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `hosting_id` INT(10) UNSIGNED NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `value` TEXT DEFAULT NULL,
  `encrypted` TINYINT(1) DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_hosting_id` (`hosting_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblhosting_config_options

Service config option values.

```sql
CREATE TABLE `tblhosting_config_options` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `hostingid` INT(10) UNSIGNED NOT NULL,
  `configid` INT(10) UNSIGNED NOT NULL,
  `optionid` INT(10) UNSIGNED DEFAULT NULL,
  `qty` INT(10) DEFAULT NULL,
  `value` VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_hostingid_configid` (`hostingid`, `configid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblhosting_history

Service change history.

```sql
CREATE TABLE `tblhosting_history` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `hostingid` INT(10) UNSIGNED NOT NULL,
  `date` DATETIME DEFAULT NULL,
  `action` VARCHAR(50) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `ipaddress` VARCHAR(45) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_hostingid` (`hostingid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblhosting_ipaddresses

Service IP addresses.

```sql
CREATE TABLE `tblhosting_ipaddresses` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `hosting_id` INT(10) UNSIGNED NOT NULL,
  `ip_address` VARCHAR(45) NOT NULL,
  `type` VARCHAR(50) DEFAULT 'main',
  `assigned_date` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_hosting_id` (`hosting_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblaffiliates

Affiliate accounts.

```sql
CREATE TABLE `tblaffiliates` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `clientid` INT(10) UNSIGNED NOT NULL,
  `code` VARCHAR(50) DEFAULT NULL,
  `date` DATETIME DEFAULT NULL,
  `status` VARCHAR(50) DEFAULT 'Active',
  `commission` DECIMAL(10,2) DEFAULT NULL,
  `balance` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `withdrawn` DECIMAL(10,2) DEFAULT 0.00,
  `paymonthly` TINYINT(1) NOT NULL DEFAULT 0,
  `noemptyaccounts` TINYINT(1) NOT NULL DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_clientid` (`clientid`),
  KEY `idx_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblaffiliatesessions

Affiliate referrals.

```sql
CREATE TABLE `tblaffiliatesessions` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `affiliateid` INT(10) UNSIGNED NOT NULL,
  `date` DATETIME DEFAULT NULL,
  `ip` VARCHAR(45) DEFAULT NULL,
  `referredby` INT(10) UNSIGNED DEFAULT NULL,
  `orderid` INT(10) UNSIGNED DEFAULT NULL,
  `amount` DECIMAL(10,2) DEFAULT NULL,
  `commission` DECIMAL(10,2) DEFAULT NULL,
  `pending` TINYINT(1) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  KEY `idx_affiliateid` (`affiliateid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblaffiliatepayouts

Affiliate payments.

```sql
CREATE TABLE `tblaffiliatepayouts` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `affiliateid` INT(10) UNSIGNED NOT NULL,
  `date` DATETIME DEFAULT NULL,
  `amount` DECIMAL(10,2) NOT NULL,
  `method` VARCHAR(50) DEFAULT NULL,
  `transactionid` VARCHAR(255) DEFAULT NULL,
  `status` VARCHAR(50) DEFAULT 'Pending',
  `notes` TEXT DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_affiliateid` (`affiliateid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Relationships

### Service -> Client
```php
$service = Capsule::table('tblhosting')
    ->select('tblhosting.*', 'tblclients.firstname', 'tblclients.lastname')
    ->join('tblclients', 'tblclients.id', '=', 'tblhosting.userid')
    ->where('tblhosting.id', $serviceId)
    ->first();
```

### Service -> Product
```php
$service = Capsule::table('tblhosting')
    ->select('tblhosting.*', 'tblproducts.name as product_name')
    ->join('tblproducts', 'tblproducts.id', '=', 'tblhosting.packageid')
    ->where('tblhosting.id', $serviceId)
    ->first();
```

## Query Examples

### Get active services with usage

```php
function getActiveServicesWithUsage(int $clientId): array
{
    return Capsule::table('tblhosting')
        ->select('tblhosting.*', 'tblproducts.name as product_name')
        ->join('tblproducts', 'tblproducts.id', '=', 'tblhosting.packageid')
        ->where('tblhosting.userid', $clientId)
        ->where('tblhosting.domainstatus', 'Active')
        ->get();
}
```

### Get services due for renewal

```php
function getServicesDueForRenewal(): array
{
    return Capsule::table('tblhosting')
        ->select('tblhosting.*', 'tblclients.email')
        ->join('tblclients', 'tblclients.id', '=', 'tblhosting.userid')
        ->where('tblhosting.nextduedate', '<=', date('Y-m-d'))
        ->where('tblhosting.domainstatus', 'Active')
        ->get();
}
```

### Get service with config options

```php
function getServiceWithConfigOptions(int $serviceId): array
{
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();
    
    $configOptions = Capsule::table('tblhosting_config_options')
        ->select('tblhosting_config_options.*', 'tblproductconfigoptions.optionname')
        ->join('tblproductconfigoptions', 'tblproductconfigoptions.id', '=', 'tblhosting_config_options.configid')
        ->where('tblhosting_config_options.hostingid', $serviceId)
        ->get();
    
    return [
        'service' => $service,
        'config_options' => $configOptions
    ];
}
```

## Related Documentation

- [whmcs-functions-services.md](../functions/whmcs-functions-services.md)
- [whmcs-schema-products.md](whmcs-schema-products.md)
- [whmcs-module-provisioning-api.md](../modules/whmcs-module-provisioning-api.md)