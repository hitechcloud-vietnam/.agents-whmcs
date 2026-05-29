# WHMCS Clients Database Schema

Complete reference for the clients table and related structures in WHMCS.

## Overview

The clients table stores all customer account information including personal details, billing information, and account status.

## Main Tables

### tblclients

Primary client/account table.

```sql
CREATE TABLE `tblclients` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `uuid` CHAR(36) DEFAULT NULL,
  `email` VARCHAR(255) NOT NULL,
  `firstname` VARCHAR(100) NOT NULL DEFAULT '',
  `lastname` VARCHAR(100) NOT NULL DEFAULT '',
  `companyname` VARCHAR(255) NOT NULL DEFAULT '',
  `phonenumber` VARCHAR(30) NOT NULL DEFAULT '',
  `password` VARCHAR(255) NOT NULL DEFAULT '',
  `salt` VARCHAR(50) DEFAULT NULL,
  `authmodule` VARCHAR(50) DEFAULT NULL,
  `authdata` TEXT DEFAULT NULL,
  `currency` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `defaultgateway` VARCHAR(50) DEFAULT NULL,
  `groupid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `language` VARCHAR(50) NOT NULL DEFAULT '',
  `sendemail` TINYINT(1) NOT NULL DEFAULT 1,
  `emailverified` TINYINT(1) NOT NULL DEFAULT 0,
  `verificationdata` TEXT DEFAULT NULL,
  `crmid` VARCHAR(255) DEFAULT NULL,
  `ip` VARCHAR(45) DEFAULT NULL,
  `host` VARCHAR(255) DEFAULT NULL,
  `status` VARCHAR(50) NOT NULL DEFAULT 'Active',
  `smsverify` TINYINT(1) NOT NULL DEFAULT 0,
  `allow_sso` TINYINT(1) NOT NULL DEFAULT 1,
  `email_optout` TINYINT(1) NOT NULL DEFAULT 0,
  `taxexempt` TINYINT(1) NOT NULL DEFAULT 0,
  `datecreated` DATE NOT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `lastlogin` DATETIME DEFAULT NULL,
  `lastupdate` DATETIME DEFAULT NULL,
  `password_last_updated` DATETIME DEFAULT NULL,
  `email_verified_at` DATETIME DEFAULT NULL,
  `credit` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `tax_id` VARCHAR(50) DEFAULT NULL,
  `billing_address1` VARCHAR(255) DEFAULT NULL,
  `billing_address2` VARCHAR(255) DEFAULT NULL,
  `billing_city` VARCHAR(100) DEFAULT NULL,
  `billing_state` VARCHAR(100) DEFAULT NULL,
  `billing_postcode` VARCHAR(20) DEFAULT NULL,
  `billing_country` VARCHAR(2) DEFAULT NULL,
  `billing_phonenumber` VARCHAR(30) DEFAULT NULL,
  `shipping_address1` VARCHAR(255) DEFAULT NULL,
  `shipping_address2` VARCHAR(255) DEFAULT NULL,
  `shipping_city` VARCHAR(100) DEFAULT NULL,
  `shipping_state` VARCHAR(100) DEFAULT NULL,
  `shipping_postcode` VARCHAR(20) DEFAULT NULL,
  `shipping_country` VARCHAR(2) DEFAULT NULL,
  `shipping_phonenumber` VARCHAR(30) DEFAULT NULL,
  `notes` TEXT DEFAULT NULL,
  `template` VARCHAR(50) DEFAULT NULL,
  `separate_invoices` TINYINT(1) NOT NULL DEFAULT 0,
  `disable_automatic_discount` TINYINT(1) NOT NULL DEFAULT 0,
  `email_verified` TINYINT(1) NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `email` (`email`),
  KEY `idx_uuid` (`uuid`),
  KEY `idx_status` (`status`),
  KEY `idx_groupid` (`groupid`),
  KEY `idx_datecreated` (`datecreated`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Field Descriptions

### Core Fields

| Field | Type | Description |
|-------|------|-------------|
| id | INT | Primary key, auto-increment |
| uuid | CHAR(36) | Unique identifier for API use |
| email | VARCHAR(255) | Primary email address (unique) |
| firstname | VARCHAR(100) | First name |
| lastname | VARCHAR(100) | Last name |
| companyname | VARCHAR(255) | Company/business name |
| phonenumber | VARCHAR(30) | Primary phone number |

### Authentication Fields

| Field | Type | Description |
|-------|------|-------------|
| password | VARCHAR(255) | Bcrypt hashed password |
| salt | VARCHAR(50) | Password salt (legacy) |
| authmodule | VARCHAR(50) | External auth module |
| authdata | TEXT | External auth data |
| emailverified | TINYINT | Email verification status |
| smsverify | TINYINT | SMS verification status |

### Preferences

| Field | Type | Description |
|-------|------|-------------|
| currency | INT | Default currency ID |
| defaultgateway | VARCHAR(50) | Default payment gateway |
| groupid | INT | Client group ID |
| language | VARCHAR(50) | Preferred language |
| template | VARCHAR(50) | Custom template |

### Financial

| Field | Type | Description |
|-------|------|-------------|
| credit | DECIMAL(10,2) | Account credit balance |
| tax_id | VARCHAR(50) | Tax identification number |
| taxexempt | TINYINT | Tax exempt status |
| separate_invoices | TINYINT | Separate invoices per service |
| disable_automatic_discount | TINYINT | Disable auto discounts |

### Timestamps

| Field | Type | Description |
|-------|------|-------------|
| datecreated | DATE | Account creation date |
| created_at | DATETIME | Full timestamp creation |
| updated_at | DATETIME | Last update timestamp |
| lastlogin | DATETIME | Last login time |
| lastupdate | DATETIME | Last data update |
| password_last_updated | DATETIME | Password change time |
| email_verified_at | DATETIME | Email verification time |

## Related Tables

### tblcontacts

Client contact/sub-account table.

```sql
CREATE TABLE `tblcontacts` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `uuid` CHAR(36) DEFAULT NULL,
  `userid` INT(10) UNSIGNED NOT NULL,
  `firstname` VARCHAR(100) NOT NULL DEFAULT '',
  `lastname` VARCHAR(100) NOT NULL DEFAULT '',
  `email` VARCHAR(255) NOT NULL,
  `companyname` VARCHAR(255) NOT NULL DEFAULT '',
  `phonenumber` VARCHAR(30) NOT NULL DEFAULT '',
  `subaccount` TINYINT(1) NOT NULL DEFAULT 0,
  `password` VARCHAR(255) DEFAULT NULL,
  `permissions` VARCHAR(255) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `lastlogin` DATETIME DEFAULT NULL,
  `email_verified` TINYINT(1) NOT NULL DEFAULT 0,
  `email_verified_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`),
  KEY `idx_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Permissions Field Values:**
- `products` - Access to products
- `invoices` - Access to invoices
- `support` - Access to support tickets
- `quotes` - Access to quotes

### tblclientgroups

Client groups for categorization.

```sql
CREATE TABLE `tblclientgroups` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `groupname` VARCHAR(255) NOT NULL DEFAULT '',
  `groupcolour` VARCHAR(20) DEFAULT NULL,
  `discount_percent` DECIMAL(5,2) DEFAULT NULL,
  `susptermexempt` TINYINT(1) NOT NULL DEFAULT 0,
  `separate_invoices` TINYINT(1) NOT NULL DEFAULT 0,
  `defaultgateway` VARCHAR(50) DEFAULT NULL,
  `sort_order` INT(10) NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblcurrencies

Currency definitions.

```sql
CREATE TABLE `tblcurrencies` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `code` VARCHAR(5) NOT NULL,
  `prefix` VARCHAR(10) DEFAULT NULL,
  `suffix` VARCHAR(10) DEFAULT NULL,
  `rate` DECIMAL(10,5) NOT NULL DEFAULT 1.00000,
  `default` TINYINT(1) NOT NULL DEFAULT 0,
  `ordering` INT(11) DEFAULT NULL,
  `base` DECIMAL(10,5) DEFAULT 1.00000,
  PRIMARY KEY (`id`),
  KEY `idx_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblcountries

Country definitions.

```sql
CREATE TABLE `tblcountries` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `iso2` CHAR(2) NOT NULL,
  `iso3` CHAR(3) DEFAULT NULL,
  `name` VARCHAR(100) NOT NULL,
  `display_name` VARCHAR(100) DEFAULT NULL,
  `phonecode` VARCHAR(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `iso2` (`iso2`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblclients_transient

Temporary client session data.

```sql
CREATE TABLE `tblclients_transient` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `client_id` INT(10) UNSIGNED NOT NULL,
  `type` VARCHAR(50) NOT NULL,
  `data` TEXT DEFAULT NULL,
  `expires` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_client_type` (`client_id`, `type`),
  KEY `idx_expires` (`expires`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Relationships

### Orders Relationship
```
tblclients (1) ──────< tblorders.userid
```

### Invoices Relationship
```
tblclients (1) ──────< tblinvoices.userid
```

### Services Relationship
```
tblclients (1) ──────< tblhosting.userid
```

### Domains Relationship
```
tblclients (1) ──────< tbldomains.userid
```

### Contacts Relationship
```
tblclients (1) ──────< tblcontacts.userid
```

### Transactions Relationship
```
tblclients (1) ──────< tbltransections.userid
```

## Query Examples

### Get client with contacts

```php
$client = Capsule::table('tblclients')
    ->where('id', 123)
    ->first();

$contacts = Capsule::table('tblcontacts')
    ->where('userid', 123)
    ->get();
```

### Search clients with group

```php
$clients = Capsule::table('tblclients')
    ->select('tblclients.*', 'tblclientgroups.groupname')
    ->leftJoin('tblclientgroups', 'tblclientgroups.id', '=', 'tblclients.groupid')
    ->where('tblclients.firstname', 'like', '%' . $search . '%')
    ->orWhere('tblclients.lastname', 'like', '%' . $search . '%')
    ->orWhere('tblclients.email', 'like', '%' . $search . '%')
    ->get();
```

### Get client statistics

```php
function getClientStats(int $clientId): array
{
    return [
        'active_services' => Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Active')
            ->count(),
        
        'pending_orders' => Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->where('status', 'Pending')
            ->count(),
        
        'open_tickets' => Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->whereIn('status', ['Open', 'Answered'])
            ->count(),
        
        'unpaid_invoices' => Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Unpaid')
            ->sum('total'),
    ];
}
```

## Indexes and Performance

### Recommended Indexes

```sql
-- For email lookups
CREATE INDEX idx_clients_email ON tblclients(email);

-- For status filtering
CREATE INDEX idx_clients_status ON tblclients(status);

-- For date-based queries
CREATE INDEX idx_clients_datecreated ON tblclients(datecreated);

-- For group-based queries
CREATE INDEX idx_clients_groupid ON tblclients(groupid);
```

### Query Optimization

```php
// Use specific columns instead of SELECT *
$clients = Capsule::table('tblclients')
    ->select('id', 'email', 'firstname', 'lastname')
    ->where('status', 'Active')
    ->get();

// Use indexes effectively
$activeClients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->where('groupid', 5)
    ->get();
```

## Related Documentation

- [whmcs-functions-clients.md](../functions/whmcs-functions-clients.md)
- [whmcs-schema-orders.md](whmcs-schema-orders.md)
- [whmcs-schema-invoices.md](whmcs-schema-invoices.md)