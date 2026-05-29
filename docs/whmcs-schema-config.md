# WHMCS Configuration Schema

Complete reference for the configuration tables in WHMCS.

## Main Tables

### tblconfiguration

System configuration.

```sql
CREATE TABLE `tblconfiguration` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `setting` VARCHAR(255) NOT NULL,
  `value` TEXT DEFAULT NULL,
  `group` VARCHAR(50) DEFAULT NULL,
  `friendlyname` VARCHAR(255) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `password` TINYINT(1) NOT NULL DEFAULT 0,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `setting` (`setting`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Common Configuration Settings

| Setting | Description |
|---------|-------------|
| Domain | System domain URL |
| systemurl | WHMCS installation URL |
| systemsslurl | HTTPS URL |
| templateserver | Template directory |
| template | Default template |
| lictype | License type |
| Version | WHMCS version |
| AllowRegister | Allow new registrations |
| AllowTransfer | Allow domain transfers |
| EncryptionKey | Data encryption key |
| MaintenanceMode | Maintenance mode status |

### tblspamfilters

Spam filter rules.

```sql
CREATE TABLE `tblspamfilters` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `type` VARCHAR(50) DEFAULT NULL,
  `prefix` VARCHAR(50) DEFAULT NULL,
  `content` TEXT DEFAULT NULL,
  `action` VARCHAR(50) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbladmin和安全

Admin security settings.

```sql
CREATE TABLE `tbladmin_security` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `setting` VARCHAR(100) NOT NULL,
  `value` VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Security Settings:**
| Setting | Description |
|---------|-------------|
| AdminSessionIdleTimeout | Session idle timeout (minutes) |
| adminLoginTimeout | Login timeout |
| min_password_strength | Minimum password strength |
| twofa_enabled | Two-factor auth requirement |
| ip_restriction | IP restriction enabled |

### tblbannedfiles

Banned file types.

```sql
CREATE TABLE `tblbannedfiles` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `extension` VARCHAR(50) NOT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `extension` (`extension`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbladminignore

Admin ignore filters.

```sql
CREATE TABLE `tbladminignore` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `type` VARCHAR(50) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblpromotions

Promotional codes.

```sql
CREATE TABLE `tblpromotions` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `code` VARCHAR(50) NOT NULL,
  `type` VARCHAR(50) DEFAULT 'percentage',
  `value` DECIMAL(10,2) DEFAULT NULL,
  `cycles` INT(10) DEFAULT NULL,
  `maxuses` INT(10) DEFAULT NULL,
  `uses` INT(10) DEFAULT NULL,
  `startdate` DATE DEFAULT NULL,
  `expirydate` DATE DEFAULT NULL,
  `lifetime` INT(10) DEFAULT NULL,
  `appliesto` TEXT DEFAULT NULL,
  `requirements` TEXT DEFAULT NULL,
  `autoapply` TINYINT(1) DEFAULT 0,
  `enabled` TINYINT(1) NOT NULL DEFAULT 1,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get configuration value

```php
function getConfigValue(string $setting, $default = null)
{
    $config = Capsule::table('tblconfiguration')
        ->where('setting', $setting)
        ->first();
    
    return $config ? $config->value : $default;
}
```

### Set configuration value

```php
function setConfigValue(string $setting, $value): bool
{
    return Capsule::table('tblconfiguration')
        ->updateOrInsert(
            ['setting' => $setting],
            [
                'value' => $value,
                'updated_at' => date('Y-m-d H:i:s')
            ]
        );
}
```

### Get all configurations

```php
function getAllConfigs(string $group = null): array
{
    $query = Capsule::table('tblconfiguration');
    
    if ($group) {
        $query->where('group', $group);
    }
    
    return $query->get()
        ->keyBy('setting')
        ->map(function($item) {
            return $item->value;
        })
        ->toArray();
}
```

## Related Documentation

- [whmcs-functions-utility.md](../functions/whmcs-functions-utility.md)