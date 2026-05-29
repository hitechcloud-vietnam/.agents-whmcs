# WHMCS Module Versioning

Complete guide for versioning WHMCS modules.

## Overview

Proper versioning ensures smooth upgrades and compatibility tracking.

## Semantic Versioning

### Version Format

```
MAJOR.MINOR.PATCH
1.5.2

- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes (backward compatible)
```

### Version Examples

```php
/**
 * Version 1.0.0 - Initial release
 */
const MODULE_VERSION = '1.0.0';

/**
 * Version 1.1.0 - Added new feature
 */
const MODULE_VERSION = '1.1.0';

/**
 * Version 2.0.0 - Breaking changes
 */
const MODULE_VERSION = '2.0.0';
```

## Version Management

### Storage

```php
<?php
/**
 * Store module version
 */
function yourmodule_setVersion(string $version): void
{
    Capsule::table('tblconfiguration')->updateOrInsert(
        ['setting' => 'YourModuleVersion'],
        ['value' => $version]
    );
}

/**
 * Get module version
 */
function yourmodule_getVersion(): string
{
    $config = Capsule::table('tblconfiguration')
        ->where('setting', 'YourModuleVersion')
        ->first();
    
    return $config ? $config->value : '0.0.0';
}

/**
 * Check if upgrade needed
 */
function yourmodule_needsUpgrade(string $currentVersion): bool
{
    return version_compare($currentVersion, MODULE_VERSION, '<');
}
```

### Version Constants

```php
<?php
/**
 * Version constants in module file
 */
namespace Vendor\YourModule;

// Module metadata
const MODULE_NAME = 'yourmodule';
const MODULE_VERSION = '1.5.0';
const MODULE_MIN_WHMCS_VERSION = '8.0';
const MODULE_MAX_WHMCS_VERSION = '8.9';

// Supported PHP versions
const MODULE_MIN_PHP_VERSION = '7.4';

// Changelog
const MODULE_CHANGELOG = [
    '1.5.0' => [
        'date' => '2024-01-15',
        'changes' => [
            'Added new API endpoint',
            'Improved error handling',
            'Fixed timezone issues',
        ],
    ],
    '1.4.0' => [
        'date' => '2023-11-20',
        'changes' => [
            'Added bulk operations support',
            'Performance improvements',
        ],
    ],
];
```

## Version Comparison

### Upgrade Path

```php
<?php
/**
 * Determine upgrade path
 */
function yourmodule_getUpgradePath(string $fromVersion, string $toVersion): array
{
    $path = [];
    $versions = ['1.0.0', '1.1.0', '1.2.0', '1.3.0', '1.4.0', '1.5.0'];
    
    foreach ($versions as $version) {
        if (version_compare($fromVersion, $version, '<')) {
            $path[] = $version;
        }
        
        if ($version === $toVersion) {
            break;
        }
    }
    
    return $path;
}

/**
 * Execute sequential upgrades
 */
function yourmodule_executeUpgradePath(array $upgradePath): bool
{
    foreach ($upgradePath as $version) {
        $result = yourmodule_upgradeToVersion($version);
        
        if (!$result['success']) {
            return false;
        }
    }
    
    return true;
}

/**
 * Upgrade to specific version
 */
function yourmodule_upgradeToVersion(string $version): array
{
    logActivity("YourModule: Starting upgrade to {$version}");
    
    switch ($version) {
        case '1.1.0':
            return yourmodule_upgrade_1_1_0();
        case '1.2.0':
            return yourmodule_upgrade_1_2_0();
        case '1.3.0':
            return yourmodule_upgrade_1_3_0();
        default:
            return ['success' => true];
    }
}
```

### Version Compatibility

```php
<?php
/**
 * Check WHMCS version compatibility
 */
function yourmodule_checkCompatibility(): array
{
    $whmcsVersion = App::getVersion();
    $minVersion = '8.0.0';
    $maxVersion = '8.9.9';
    
    if (version_compare($whmcsVersion, $minVersion, '<')) {
        return [
            'compatible' => false,
            'reason' => "WHMCS version {$whmcsVersion} is below minimum required ({$minVersion})",
        ];
    }
    
    if (version_compare($whmcsVersion, $maxVersion, '>')) {
        return [
            'compatible' => false,
            'reason' => "WHMCS version {$whmcsVersion} exceeds maximum supported ({$maxVersion})",
        ];
    }
    
    return [
        'compatible' => true,
        'whmcs_version' => $whmcsVersion,
    ];
}

/**
 * Check PHP version compatibility
 */
function yourmodule_checkPhpCompatibility(): array
{
    $phpVersion = PHP_VERSION;
    $minPhpVersion = '7.4.0';
    
    if (version_compare($phpVersion, $minPhpVersion, '<')) {
        return [
            'compatible' => false,
            'reason' => "PHP version {$phpVersion} is below minimum required ({$minPhpVersion})",
        ];
    }
    
    return [
        'compatible' => true,
        'php_version' => $phpVersion,
    ];
}
```

## Changelog Management

### Changelog Format

```markdown
# Changelog

All notable changes to this module will be documented in this file.

## [1.5.0] - 2024-01-15

### Added
- New API endpoint for bulk operations
- Support for custom metadata fields
- Webhook notifications for account changes

### Changed
- Improved API response handling
- Optimized database queries
- Updated documentation

### Fixed
- Timezone conversion issue
- Missing error codes in responses
- Session timeout handling

### Security
- Updated API authentication
- Added input sanitization

## [1.4.0] - 2023-11-20

### Added
- Initial release of advanced features
- Multi-server support
```

### Automated Changelog

```php
<?php
/**
 * Generate changelog entry
 */
function yourmodule_generateChangelog(string $version, array $changes): string
{
    $date = date('Y-m-d');
    $entry = "## [{$version}] - {$date}\n\n";
    
    foreach ($changes as $type => $items) {
        $heading = ucfirst($type);
        $entry .= "### {$heading}\n";
        
        foreach ($items as $item) {
            $entry .= "- {$item}\n";
        }
        
        $entry .= "\n";
    }
    
    return $entry;
}
```

## Version Transitions

### Major Version Upgrade

```php
<?php
/**
 * Major version upgrade (v1.x -> v2.x)
 */
function yourmodule_upgrade_v2_0_0(): array
{
    try {
        // Database migration required
        if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'metadata')) {
            Capsule::schema()->table('mod_yourmodule_data', function($table) {
                $table->json('metadata')->nullable();
            });
        }
        
        // Migrate existing data
        $existingRecords = Capsule::table('mod_yourmodule_data')
            ->whereNull('metadata')
            ->get();
        
        foreach ($existingRecords as $record) {
            Capsule::table('mod_yourmodule_data')
                ->where('id', $record->id)
                ->update([
                    'metadata' => json_encode([
                        'legacy_id' => $record->account_id,
                        'migrated_at' => date('Y-m-d H:i:s'),
                    ]),
                ]);
        }
        
        // Update deprecated settings
        Capsule::table('tblconfiguration')
            ->where('setting', 'YourModuleOldSetting')
            ->update(['setting' => 'YourModuleNewSetting']);
        
        logActivity('YourModule: Completed major upgrade to v2.0.0');
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => 'Upgrade failed: ' . $e->getMessage(),
        ];
    }
}
```

### Rollback Support

```php
<?php
/**
 * Rollback to previous version
 */
function yourmodule_rollback(string $targetVersion): array
{
    try {
        // Create rollback point
        $currentVersion = yourmodule_getVersion();
        Capsule::table('mod_yourmodule_settings')->insert([
            'setting_key' => 'rollback_point',
            'setting_value' => json_encode([
                'from_version' => $currentVersion,
                'target_version' => $targetVersion,
                'created_at' => date('Y-m-d H:i:s'),
            ]),
        ]);
        
        // Perform rollback based on target version
        switch ($targetVersion) {
            case '1.4.0':
                rollbackToV1_4_0();
                break;
            case '1.3.0':
                rollbackToV1_3_0();
                break;
        }
        
        yourmodule_setVersion($targetVersion);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => 'Rollback failed: ' . $e->getMessage(),
        ];
    }
}
```

## Best Practices

1. **Follow semantic versioning** - Use MAJOR.MINOR.PATCH format
2. **Document breaking changes** - Clearly note incompatible changes
3. **Test all upgrade paths** - Verify transitions between versions
4. **Provide rollback capability** - Allow reverting failed upgrades
5. **Keep changelog updated** - Document all changes
6. **Check compatibility** - Verify WHMCS and PHP versions

## Related Documentation

- [whmcs-module-lifecycle.md](whmcs-module-lifecycle.md)
- [whmcs-module-upgrade.md](whmcs-module-upgrade.md)
