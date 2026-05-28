# Module Versioning Best Practices

Versioning your WHMCS module properly is essential for maintaining compatibility, managing updates, and providing a clear upgrade path for your users.

## Semantic Versioning (SemVer)

Follow Semantic Versioning (SemVer) with format `MAJOR.MINOR.PATCH`:

- **MAJOR**: Incompatible API changes
- **MINOR**: New functionality, backward compatible
- **PATCH**: Backward compatible bug fixes

Example: Version `2.3.1` indicates major version 2, minor version 3, patch 1.

## Version Declaration in Module

### Standard Module Version

```php
<?php
/**
 * Module Version Information
 *
 * @package WHMCS Module Name
 * @version 2.3.1
 */

if (!defined("WHMCS")) {
    die("Access denied");
}

function moduleversion_MetaData()
{
    return [
        'DisplayName' => 'Module Name',
        'Version' => '2.3.1',
        'MinimumVersion' => '8.0',
        'RequiredFrameworkVersion' => '8.0.0',
        'License' => 'proprietary',
        'ChangeLog' => [
            '2.3.1' => 'Fixed cache invalidation issue',
            '2.3.0' => 'Added bulk operations support',
            '2.2.0' => 'Performance improvements',
        ],
    ];
}
```

### Version Comparison Utility

```php
<?php
/**
 * Version comparison utility for WHMCS modules
 */
class VersionComparator
{
    /**
     * Compare two version strings
     *
     * @param string $version1
     * @param string $version2
     * @return int -1, 0, or 1
     */
    public static function compare(string $version1, string $version2): int
    {
        $v1parts = self::parseVersion($version1);
        $v2parts = self::parseVersion($version2);

        for ($i = 0; $i < 3; $i++) {
            $v1 = $v1parts[$i] ?? 0;
            $v2 = $v2parts[$i] ?? 0;

            if ($v1 < $v2) {
                return -1;
            }
            if ($v1 > $v2) {
                return 1;
            }
        }

        return 0;
    }

    /**
     * Check if version meets minimum requirement
     */
    public static function meetsMinimum(string $version, string $minimum): bool
    {
        return self::compare($version, $minimum) >= 0;
    }

    private static function parseVersion(string $version): array
    {
        return array_map('intval', explode('.', $version));
    }
}
```

## Version Compatibility Matrix

| Module Version | WHMCS 7.x | WHMCS 8.0 | WHMCS 8.x |
|---------------|-----------|------------|-----------|
| 1.x           | Yes       | No         | No        |
| 2.0+          | No        | Yes        | Yes       |
| 3.0+          | No        | No         | Yes       |

## Migration Path Documentation

### Upgrade Path Example

```php
<?php
/**
 * Version-specific migration handler
 */
class MigrationHandler
{
    private $currentVersion;

    public function __construct(string $currentVersion)
    {
        $this->currentVersion = $currentVersion;
    }

    /**
     * Get required migrations from current to target version
     */
    public function getMigrations(string $targetVersion): array
    {
        $migrations = [];
        $versions = ['1.0.0', '1.1.0', '1.2.0', '2.0.0', '2.1.0', '2.3.1'];

        $startIndex = array_search($this->currentVersion, $versions) ?: 0;
        $endIndex = array_search($targetVersion, $versions);

        if ($endIndex === false) {
            throw new InvalidArgumentException("Unknown version: $targetVersion");
        }

        for ($i = $startIndex + 1; $i <= $endIndex; $i++) {
            $migrations[] = [
                'from' => $versions[$i - 1],
                'to' => $versions[$i],
                'file' => "migrations/migrate_{$versions[$i - 1]}_to_{$versions[$i]}.php",
            ];
        }

        return $migrations;
    }

    /**
     * Execute all pending migrations
     */
    public function migrate(string $targetVersion): MigrationResult
    {
        $migrations = $this->getMigrations($targetVersion);
        $results = [];

        foreach ($migrations as $migration) {
            $result = $this->runMigration($migration);
            $results[] = $result;

            if (!$result->success) {
                return new MigrationResult(false, $results);
            }
        }

        return new MigrationResult(true, $results);
    }

    private function runMigration(array $migration): SingleMigrationResult
    {
        // Migration execution logic
        $migrationFile = $migration['file'];

        if (!file_exists($migrationFile)) {
            return new SingleMigrationResult(
                false,
                "Migration file not found: $migrationFile"
            );
        }

        // Include and run migration
        // ... execution code ...

        return new SingleMigrationResult(true, "Migrated to {$migration['to']}");
    }
}
```

## Changelog Management

### Recommended Changelog Format

```markdown
# Changelog

All notable changes to this module are documented in this file.

## [2.3.1] - 2024-01-15

### Fixed
- Cache invalidation not triggering on product updates
- Race condition in background job processing

## [2.3.0] - 2024-01-01

### Added
- Bulk operations API for processing multiple items
- New hook: OnModuleBulkOperation

### Changed
- Improved database query performance by 40%
- Updated third-party library dependencies

### Deprecated
- `processOrder()` method (use `processBulkOrders()` instead)

## [2.2.0] - 2023-12-01

### Security
- Sanitized all user inputs
- Added CSRF token validation

### Fixed
- Memory leak in long-running processes
```

## Version-Specific Feature Flags

```php
<?php
/**
 * Feature flags based on module version
 */
class FeatureFlags
{
    private static $flags = [
        'bulk_operations' => '2.3.0',
        'advanced_caching' => '2.2.0',
        'webhook_support' => '2.0.0',
        'async_processing' => '1.5.0',
    ];

    /**
     * Check if a feature is available
     */
    public static function isAvailable(string $feature, string $currentVersion): bool
    {
        if (!isset(self::$flags[$feature])) {
            return false;
        }

        return VersionComparator::compare($currentVersion, self::$flags[$feature]) >= 0;
    }

    /**
     * Get all features available in a version
     */
    public static function getAvailableFeatures(string $version): array
    {
        $available = [];

        foreach (self::$flags as $feature => $requiredVersion) {
            if (VersionComparator::compare($version, $requiredVersion) >= 0) {
                $available[] = $feature;
            }
        }

        return $available;
    }
}
```

## Best Practices Summary

1. **Always use semantic versioning** - Makes upgrade paths predictable
2. **Document breaking changes** - Clearly mark incompatible changes
3. **Maintain changelogs** - Keep detailed records of all changes
4. **Test upgrade paths** - Verify migrations work correctly
5. **Version your database schema** - Track schema changes alongside code
6. **Support multiple WHMCS versions** - Use feature flags for compatibility
7. **Publish upgrade guides** - Help users transition between major versions

## Related Patterns

- [Database Migrations](./database-migrations.md) - Schema version management
- [Module Packaging](./module-packaging.md) - Distributing versioned modules
- [Service Layer](./service-layer.md) - Modular service architecture