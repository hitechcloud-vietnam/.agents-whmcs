# WHMCS Module Configuration API

Complete reference for managing module configuration in WHMCS.

## Configuration Storage

### Save Configuration

```php
/**
 * Save module configuration
 */
function saveModuleConfig(string $moduleType, string $moduleName, array $config): bool
{
    foreach ($config as $key => $value) {
        Capsule::table('tblmodules')->updateOrInsert(
            ['type' => $moduleType, 'name' => $moduleName, 'setting' => $key],
            ['value' => is_array($value) ? json_encode($value) : $value]
        );
    }
    
    return true;
}
```

### Get Configuration

```php
/**
 * Get module configuration
 */
function getModuleConfig(string $moduleType, string $moduleName): array
{
    return Capsule::table('tblmodules')
        ->where('type', $moduleType)
        ->where('name', $moduleName)
        ->pluck('value', 'setting')
        ->toArray();
}
```

### Delete Configuration

```php
/**
 * Delete module configuration
 */
function deleteModuleConfig(string $moduleType, string $moduleName): bool
{
    return Capsule::table('tblmodules')
        ->where('type', $moduleType)
        ->where('name', $moduleName)
        ->delete() > 0;
}
```

## Server Configuration

### Save Server Config

```php
/**
 * Save server module configuration
 */
function saveServerConfig(int $serverId, array $config): bool
{
    foreach ($config as $key => $value) {
        Capsule::table('tblservers')->updateOrInsert(
            ['id' => $serverId, 'setting' => $key],
            ['value' => $value]
        );
    }
    
    return true;
}
```

### Get Server Config

```php
/**
 * Get server module configuration
 */
function getServerConfig(int $serverId): array
{
    return Capsule::table('tblservers')
        ->where('id', $serverId)
        ->pluck('value', 'setting')
        ->toArray();
}
```

## Related Documentation

- [whmcs-schema-modules.md](../schema/whmcs-schema-modules.md)