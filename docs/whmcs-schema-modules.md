# WHMCS Modules Database Schema

Complete reference for module management tables in WHMCS.

## Main Tables

### tblmodules

Module definitions.

```sql
CREATE TABLE `tblmodules` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `type` VARCHAR(50) NOT NULL,
  `name` VARCHAR(100) NOT NULL,
  `value` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_type` (`type`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Module Types

| Type | Description |
|------|-------------|
| server | Server provisioning |
| registrar | Domain registrar |
| gateway | Payment gateway |
| addon | Addon module |
| notification | Notification provider |

### tblservers

Server definitions.

```sql
CREATE TABLE `tblservers` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `ip` VARCHAR(45) DEFAULT NULL,
  `hostname` VARCHAR(255) DEFAULT NULL,
  `status` VARCHAR(50) DEFAULT 'Active',
  `maxaccounts` INT(10) DEFAULT NULL,
  `type` VARCHAR(50) DEFAULT NULL,
  `username` VARCHAR(255) DEFAULT NULL,
  `password` VARCHAR(255) DEFAULT NULL,
  `accesshash` TEXT DEFAULT NULL,
  `secure` TINYINT(1) DEFAULT 0,
  `nameserver` VARCHAR(255) DEFAULT NULL,
  `nameserver2` VARCHAR(255) DEFAULT NULL,
  `nameserver3` VARCHAR(255) DEFAULT NULL,
  `nameserverport` INT(10) DEFAULT NULL,
  `disabled` TINYINT(1) DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblservergroups

Server groups.

```sql
CREATE TABLE `tblservergroups` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblserver_group_relation

Server to group mapping.

```sql
CREATE TABLE `tblserver_group_relation` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `server_id` INT(10) UNSIGNED NOT NULL,
  `group_id` INT(10) UNSIGNED NOT NULL,
  `priority` INT(10) DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_server_id` (`server_id`),
  KEY `idx_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblmodule_queue

Module processing queue.

```sql
CREATE TABLE `tblmodule_queue` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `module_type` VARCHAR(50) NOT NULL,
  `module_name` VARCHAR(100) NOT NULL,
  `service_type` VARCHAR(50) DEFAULT NULL,
  `service_id` INT(10) UNSIGNED DEFAULT NULL,
  `action` VARCHAR(100) NOT NULL,
  `data` TEXT DEFAULT NULL,
  `status` VARCHAR(50) DEFAULT 'pending',
  `priority` INT(10) DEFAULT 0,
  `attempts` INT(10) DEFAULT 0,
  `max_attempts` INT(10) DEFAULT 3,
  `last_attempt` DATETIME DEFAULT NULL,
  `error` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `scheduled_at` DATETIME DEFAULT NULL,
  `completed_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_status` (`status`),
  KEY `idx_module` (`module_type`, `module_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get server by ID

```php
function getServer(int $serverId): ?object
{
    return Capsule::table('tblservers')
        ->where('id', $serverId)
        ->first();
}
```

### Get servers in group

```php
function getServersInGroup(int $groupId): array
{
    return Capsule::table('tblservers')
        ->select('tblservers.*')
        ->join('tblserver_group_relation', 'tblserver_group_relation.server_id', '=', 'tblservers.id')
        ->where('tblserver_group_relation.group_id', $groupId)
        ->where('tblservers.disabled', 0)
        ->orderBy('tblserver_group_relation.priority', 'asc')
        ->get();
}
```

### Get module config

```php
function getModuleConfig(string $type, string $name): array
{
    return Capsule::table('tblmodules')
        ->where('type', $type)
        ->where('name', $name)
        ->pluck('value', 'setting')
        ->toArray();
}
```

## Related Documentation

- [whmcs-module-provisioning-api.md](../modules/whmcs-module-provisioning-api.md)
- [whmcs-module-lifecycle.md](../modules/whmcs-module-lifecycle.md)