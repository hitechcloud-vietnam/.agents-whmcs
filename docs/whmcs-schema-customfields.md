# WHMCS Custom Fields Schema

Complete reference for custom field tables in WHMCS.

## Main Tables

### tblcustomfields

Custom field definitions.

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

### Field Types

| Type | Description |
|------|-------------|
| text | Single line text |
| textarea | Multi-line text |
| password | Password field |
| dropdown | Dropdown selection |
| multiselect | Multiple selection |
| checkbox | Yes/No checkbox |
| radio | Radio button selection |
| link | URL link |

### tblcustomfieldsvalues

Custom field values.

```sql
CREATE TABLE `tblcustomfieldsvalues` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `fieldid` INT(10) UNSIGNED NOT NULL,
  `relid` INT(10) UNSIGNED NOT NULL,
  `value` TEXT DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_fieldid_relid` (`fieldid`, `relid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get product custom fields

```php
function getProductCustomFields(int $productId): array
{
    return Capsule::table('tblcustomfields')
        ->where('relid', $productId)
        ->where('type', 'product')
        ->orderBy('sortorder', 'asc')
        ->get();
}
```

### Get client custom fields

```php
function getClientCustomFields(int $clientId): array
{
    return Capsule::table('tblcustomfields')
        ->where('type', 'client')
        ->get();
}
```

### Save custom field value

```php
function saveCustomFieldValue(int $fieldId, int $relId, $value): bool
{
    return Capsule::table('tblcustomfieldsvalues')
        ->updateOrInsert(
            ['fieldid' => $fieldId, 'relid' => $relId],
            ['value' => is_array($value) ? json_encode($value) : $value]
        );
}
```

## Related Documentation

- [whmcs-schema-products.md](whmcs-schema-products.md)
- [whmcs-schema-clients.md](whmcs-schema-clients.md)