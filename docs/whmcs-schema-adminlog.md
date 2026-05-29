# WHMCS Admin Log Schema

Complete reference for the admin activity log table in WHMCS.

## Main Tables

### tbladminlog

Admin activity log.

```sql
CREATE TABLE `tbladminlog` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `adminid` INT(10) UNSIGNED NOT NULL,
  `loglevel` VARCHAR(20) DEFAULT 'info',
  `description` TEXT DEFAULT NULL,
  `extrainfo` TEXT DEFAULT NULL,
  `ipaddress` VARCHAR(45) DEFAULT NULL,
  `date` DATETIME DEFAULT NULL,
  `user_agent` VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_adminid` (`adminid`),
  KEY `idx_date` (`date`),
  KEY `idx_loglevel` (`loglevel`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Log Levels

| Level | Description |
|-------|-------------|
| debug | Debug information |
| info | General information |
| notice | Important notice |
| warning | Warning |
| error | Error occurred |
| critical | Critical issue |

### tbladmins

Admin users.

```sql
CREATE TABLE `tbladmins` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `uuid` CHAR(36) DEFAULT NULL,
  `username` VARCHAR(100) NOT NULL,
  `password` VARCHAR(255) NOT NULL,
  `authmodule` VARCHAR(50) DEFAULT NULL,
  `authdata` TEXT DEFAULT NULL,
  `email` VARCHAR(255) NOT NULL,
  `firstname` VARCHAR(100) DEFAULT NULL,
  `lastname` VARCHAR(100) DEFAULT NULL,
  `signature` TEXT DEFAULT NULL,
  `permissions` TEXT DEFAULT NULL,
  `disabled` TINYINT(1) NOT NULL DEFAULT 0,
  `两根factor_auth` TINYINT(1) NOT NULL DEFAULT 0,
  `twofa_secret` VARCHAR(255) DEFAULT NULL,
  `logouttime` INT(10) DEFAULT NULL,
  `lastlogin` DATETIME DEFAULT NULL,
  `lastlogout` DATETIME DEFAULT NULL,
  `lastactivity` DATETIME DEFAULT NULL,
  `language` VARCHAR(50) DEFAULT NULL,
  `email_verified` TINYINT(1) NOT NULL DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `username` (`username`),
  KEY `idx_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbladmin_roles

Admin role definitions.

```sql
CREATE TABLE `tbladmin_roles` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `permissions` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get admin activity

```php
function getAdminActivity(int $adminId, int $limit = 100): array
{
    return Capsule::table('tbladminlog')
        ->where('adminid', $adminId)
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
}
```

### Get failed admin actions

```php
function getFailedAdminActions(int $limit = 50): array
{
    return Capsule::table('tbladminlog')
        ->select('tbladminlog.*', 'tbladmins.username')
        ->join('tbladmins', 'tbladmins.id', '=', 'tbladminlog.adminid')
        ->whereIn('loglevel', ['error', 'critical'])
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
}
```

### Log admin action

```php
function logAdminAction(
    int $adminId,
    string $description,
    string $level = 'info',
    ?string $extraInfo = null
): int {
    return Capsule::table('tbladminlog')->insertGetId([
        'adminid' => $adminId,
        'description' => $description,
        'loglevel' => $level,
        'extrainfo' => $extraInfo,
        'ipaddress' => $_SERVER['REMOTE_ADDR'] ?? null,
        'date' => date('Y-m-d H:i:s'),
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? null
    ]);
}
```

## Related Documentation

- [whmcs-functions-logging.md](../functions/whmcs-functions-logging.md)