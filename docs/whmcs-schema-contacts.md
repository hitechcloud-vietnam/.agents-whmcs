# WHMCS Contacts Database Schema

Complete reference for client contacts tables in WHMCS.

## Main Tables

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
  `address1` VARCHAR(255) DEFAULT NULL,
  `address2` VARCHAR(255) DEFAULT NULL,
  `city` VARCHAR(100) DEFAULT NULL,
  `state` VARCHAR(100) DEFAULT NULL,
  `postcode` VARCHAR(20) DEFAULT NULL,
  `country` VARCHAR(2) DEFAULT NULL,
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

### Contact Permissions

| Permission | Description |
|------------|-------------|
| products | Access to products/services |
| invoices | Access to invoices |
| support | Access to support tickets |
| quotes | Access to quotes |
| billing | Access to billing info |

### tblcontact_permissions

Contact permission assignments.

```sql
CREATE TABLE `tblcontact_permissions` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `contact_id` INT(10) UNSIGNED NOT NULL,
  `permission` VARCHAR(50) NOT NULL,
  `granted_by` INT(10) UNSIGNED DEFAULT NULL,
  `granted_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_contact_id` (`contact_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get client contacts

```php
function getClientContacts(int $clientId): array
{
    return Capsule::table('tblcontacts')
        ->where('userid', $clientId)
        ->get();
}
```

### Create contact

```php
function createContact(int $clientId, array $data): int
{
    return Capsule::table('tblcontacts')->insertGetId([
        'userid' => $clientId,
        'firstname' => $data['firstname'],
        'lastname' => $data['lastname'],
        'email' => $data['email'],
        'companyname' => $data['companyname'] ?? '',
        'phonenumber' => $data['phonenumber'] ?? '',
        'subaccount' => $data['subaccount'] ?? 0,
        'password' => !empty($data['password']) ? hashPassword($data['password']) : null,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}
```

## Related Documentation

- [whmcs-schema-clients.md](whmcs-schema-clients.md)