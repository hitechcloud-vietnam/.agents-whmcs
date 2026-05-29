# WHMCS Registrars Database Schema

Complete reference for domain registrar module tables in WHMCS.

## Main Tables

### tblregistrars

Registrar module configuration.

```sql
CREATE TABLE `tblregistrars` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(50) NOT NULL,
  `setting` VARCHAR(100) NOT NULL,
  `value` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldomain_registrars

Domain registrar assignments.

```sql
CREATE TABLE `tbldomain_registrars` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `domain_id` INT(10) UNSIGNED NOT NULL,
  `registrar` VARCHAR(50) NOT NULL,
  `registrar_domain_id` VARCHAR(255) DEFAULT NULL,
  `status` VARCHAR(50) DEFAULT NULL,
  `synced_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_domain_id` (`domain_id`),
  KEY `idx_registrar` (`registrar`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Registrar Settings

| Setting | Description |
|---------|-------------|
| RegistrarRequiredFields | Required fields |
| AutoRegistration | Auto-registration enabled |
| TransferAutoProvision | Auto-provision transfers |
| DefaultNameservers | Default nameservers |
| DNSManagement | DNS management available |
| EmailForwarding | Email forwarding available |
| IDProtection | ID protection available |

## Query Examples

### Get registrar config

```php
function getRegistrarConfig(string $registrar): array
{
    return Capsule::table('tblregistrars')
        ->where('name', $registrar)
        ->pluck('value', 'setting')
        ->toArray();
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

## Related Documentation

- [whmcs-module-registrar-api.md](../modules/whmcs-module-registrar-api.md)
- [whmcs-functions-domains.md](../functions/whmcs-functions-domains.md)