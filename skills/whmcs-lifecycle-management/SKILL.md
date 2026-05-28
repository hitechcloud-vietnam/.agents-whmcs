# WHMCS Module Lifecycle Management Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for managing module lifecycle from development to end-of-life.

## When to Use

- Module versioning
- Migration management
- Deprecation planning

## Lifecycle Patterns

```php
<?php
// module.json
{
    "id": "module_name",
    "version": "1.2.0",
    "whmcs_version": "8.0",
    "php_version": "8.1",
    "deprecated": false,
    "end_of_life": null,
    "support_until": "2025-12-31"
}
```

### Version Management

```php
class VersionManager {
    public function checkCompatibility(string $moduleVersion): bool {
        $minVersion = '1.0.0';
        return version_compare($moduleVersion, $minVersion, '>=');
    }

    public function getMigrationPath(string $from, string $to): array {
        $migrations = [];
        $versions = ['1.0.0', '1.1.0', '1.2.0'];

        $fromIdx = array_search($from, $versions);
        $toIdx = array_search($to, $versions);

        for ($i = $fromIdx + 1; $i <= $toIdx; $i++) {
            $migrations[] = 'MigrateTo' . str_replace('.', '', $versions[$i]);
        }

        return $migrations;
    }
}
```

### Deprecation Handling

```php
function {module}_deprecatedFunction(array $params): string {
    trigger_error(
        '{Module}::deprecatedFunction is deprecated. Use newFunction() instead.',
        E_USER_DEPRECATED
    );

    return newFunction($params);
}
```

---

**Related Skills:**
- whmcs-upgrade-handler
- whmcs-migration-guide
- whmcs-deployment
