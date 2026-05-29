# WHMCS Domains Database Schema

Complete reference for the domains table and related structures in WHMCS.

## Overview

The domains tables manage all domain registrations, transfers, and renewals.

## Main Tables

### tbldomains

Primary domains table.

```sql
CREATE TABLE `tbldomains` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `userid` INT(10) UNSIGNED NOT NULL,
  `orderid` INT(10) UNSIGNED DEFAULT NULL,
  `domain` VARCHAR(255) NOT NULL,
  `registrationdate` DATE DEFAULT NULL,
  `nextduedate` DATE DEFAULT NULL,
  `nextinvoicedate` DATE DEFAULT NULL,
  `expirydate` DATE DEFAULT NULL,
  `domainstatus` VARCHAR(50) NOT NULL DEFAULT 'Pending',
  `registrationperiod` INT(10) NOT NULL DEFAULT 1,
  `dns管理` TINYINT(1) NOT NULL DEFAULT 0,
  `emailforwarding` TINYINT(1) NOT NULL DEFAULT 0,
  `idprotection` TINYINT(1) NOT NULL DEFAULT 0,
  `registrar` VARCHAR(50) DEFAULT NULL,
  `registrationdata` TEXT DEFAULT NULL,
  `recurringamount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `paymentmethod` VARCHAR(50) DEFAULT NULL,
  `subscriptionid` VARCHAR(255) DEFAULT NULL,
  `type` VARCHAR(50) DEFAULT 'Register',
  `recurring` TINYINT(1) NOT NULL DEFAULT 1,
  `synced` TINYINT(1) NOT NULL DEFAULT 0,
  `lastcheck` DATETIME DEFAULT NULL,
  `nextcheck` DATETIME DEFAULT NULL,
  `notes` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `termination_date` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`),
  KEY `idx_domain` (`domain`(255)),
  KEY `idx_domainstatus` (`domainstatus`),
  KEY `idx_registrar` (`registrar`),
  KEY `idx_expirydate` (`expirydate`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Domain Statuses:**
| Status | Description |
|--------|-------------|
| Pending | Awaiting registration/transfer |
| Active | Domain active |
| Pending Transfer | Transfer in progress |
| Transferred | Domain transferred away |
| Expired | Domain expired |
| Cancelled | Domain cancelled |
| Suspended | Domain suspended |

**Domain Types:**
| Type | Description |
|------|-------------|
| Register | New registration |
| Transfer | Domain transfer |
| Own | Already owned domain |

### tbldomains_transfers

Domain transfer tracking.

```sql
CREATE TABLE `tbldomains_transfers` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `domain_id` INT(10) UNSIGNED NOT NULL,
  `transfer_secret` VARCHAR(255) DEFAULT NULL,
  `transfer_status` VARCHAR(50) DEFAULT NULL,
  `initiated_at` DATETIME DEFAULT NULL,
  `completed_at` DATETIME DEFAULT NULL,
  `expires_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_domain_id` (`domain_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldomainpricing

TLD pricing table.

```sql
CREATE TABLE `tbldomainpricing` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `extension` VARCHAR(50) NOT NULL,
  `autoreg` VARCHAR(255) DEFAULT NULL,
  ` GracePeriodDays` INT(10) DEFAULT NULL,
  `redemptionGracePeriodDays` INT(10) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `extension` (`extension`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldomainpricing Pricing

```sql
CREATE TABLE `tbldomainpricing` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `extension` VARCHAR(50) NOT NULL,
  `autoreg` VARCHAR(255) DEFAULT NULL,
  `dnsmanagement` DECIMAL(10,2) DEFAULT NULL,
  `emailforwarding` DECIMAL(10,2) DEFAULT NULL,
  `idprotection` DECIMAL(10,2) DEFAULT NULL,
  ` GracePeriodDays` INT(10) DEFAULT NULL,
  `redemptionGracePeriodDays` INT(10) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldomaincontacts

Domain contact information.

```sql
CREATE TABLE `tbldomaincontacts` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `domainid` INT(10) UNSIGNED NOT NULL,
  `contacttype` VARCHAR(50) NOT NULL,
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
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_domainid` (`domainid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Contact Types:**
- `registrant` - Registrant contact
- `admin` - Administrative contact
- `tech` - Technical contact
- `billing` - Billing contact

### tbldomaindnsrecords

DNS records for domains.

```sql
CREATE TABLE `tbldomaindnsrecords` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `domain_id` INT(10) UNSIGNED NOT NULL,
  `name` VARCHAR(255) DEFAULT NULL,
  `type` VARCHAR(10) DEFAULT NULL,
  `priority` INT(10) DEFAULT 0,
  `address` VARCHAR(255) DEFAULT NULL,
  `ttl` INT(10) DEFAULT 14400,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_domain_id` (`domain_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldomainnameservers

Domain nameservers.

```sql
CREATE TABLE `tbldomainnameservers` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `domain_id` INT(10) UNSIGNED NOT NULL,
  `nameserver` VARCHAR(255) NOT NULL,
  `ip` VARCHAR(45) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_domain_id` (`domain_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldomaintransferlocks

Domain transfer lock status.

```sql
CREATE TABLE `tbldomaintransferlocks` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `domain_id` INT(10) UNSIGNED NOT NULL,
  `locked` TINYINT(1) DEFAULT 0,
  `locked_at` DATETIME DEFAULT NULL,
  `unlock_requested_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_domain_id` (`domain_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Relationships

### Domain -> Client
```php
$domain = Capsule::table('tbldomains')
    ->select('tbldomains.*', 'tblclients.firstname', 'tblclients.lastname')
    ->join('tblclients', 'tblclients.id', '=', 'tbldomains.userid')
    ->where('tbldomains.id', $domainId)
    ->first();
```

### Domain -> Contacts
```php
$contacts = Capsule::table('tbldomaincontacts')
    ->where('domainid', $domainId)
    ->get();
```

## Query Examples

### Get domains expiring soon

```php
function getDomainsExpiringSoon(int $days = 30): array
{
    return Capsule::table('tbldomains')
        ->select('tbldomains.*', 'tblclients.email')
        ->join('tblclients', 'tblclients.id', '=', 'tbldomains.userid')
        ->where('tbldomains.domainstatus', 'Active')
        ->where('tbldomains.expirydate', '<=', date('Y-m-d', strtotime("+{$days} days")))
        ->get();
}
```

### Get domains by registrar

```php
function getDomainsByRegistrar(string $registrar): array
{
    return Capsule::table('tbldomains')
        ->where('registrar', $registrar)
        ->get();
}
```

### Get domain with all details

```php
function getDomainDetails(int $domainId): array
{
    $domain = Capsule::table('tbldomains')
        ->where('id', $domainId)
        ->first();
    
    $contacts = Capsule::table('tbldomaincontacts')
        ->where('domainid', $domainId)
        ->get();
    
    $dnsRecords = Capsule::table('tbldomaindnsrecords')
        ->where('domain_id', $domainId)
        ->get();
    
    $nameservers = Capsule::table('tbldomainnameservers')
        ->where('domain_id', $domainId)
        ->get();
    
    return [
        'domain' => $domain,
        'contacts' => $contacts,
        'dns_records' => $dnsRecords,
        'nameservers' => $nameservers
    ];
}
```

## Related Documentation

- [whmcs-functions-domains.md](../functions/whmcs-functions-domains.md)
- [whmcs-module-registrar-api.md](../modules/whmcs-module-registrar-api.md)