# WHMCS Module Versioning Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers version management for WHMCS modules, including semantic versioning, version compatibility, upgrade paths, and changelog management.

---

## Semantic Versioning

### Version Format

```
MAJOR.MINOR.PATCH
     |      |    |
     |      |    +-- Patch: Bug fixes, small changes
     |      +------- Minor: New features, backwards compatible
     +-------------- Major: Breaking changes
```

### Examples

```
1.0.0 - Initial release
1.0.1 - Bug fix release
1.1.0 - Feature release (backwards compatible)
2.0.0 - Breaking change release
```

## Version Declaration

### Module MetaData

```php
/**
 * Module information with version
 */
function your_module_MetaData()
{
    return [
        'DisplayName'    => 'Your Module',
        'Author'          => 'Your Name',
        'Version'         => '1.2.0',
        'Min WHMCS Version' => '8.0.0',
        'Max WHMCS Version' => '8.9.9',
        'Language'        => 'english',
        'Release Data'    => '2026-05-28',
        
        // Additional version info
        'Requires'        => [
            'Php'  => '7.4',
        ],
    ];
}
```

### Gateway Version

```php
/**
 * Gateway API version
 */
function your_gateway_MetaData()
{
    return [
        'DisplayName' => 'Your Gateway',
        'APIVersion' => '1.1',  // Gateway API version
        'Version'    => '1.2.0', // Module version
    ];
}
```

## Upgrade Handling

### Version Check on Load

```php
/**
 * Check stored version and upgrade if needed
 */
function your_module_checkVersion()
{
    $currentVersion = get_config_var('your_module_version');
    $moduleVersion = '1.2.0';
    
    if (version_compare($currentVersion, $moduleVersion, '<')) {
        // Perform upgrade
        your_module_upgrade($currentVersion, $moduleVersion);
    }
}

/**
 * Upgrade handler
 * 
 * @param string $fromVersion Current version
 * @param string $toVersion Target version
 */
function your_module_upgrade($fromVersion, $toVersion)
{
    $versions = ['1.0.0', '1.0.1', '1.1.0', '1.1.1', '1.2.0'];
    
    foreach ($versions as $version) {
        if (version_compare($fromVersion, $version, '<') && 
            version_compare($version, $toVersion, '<=')) {
            
            upgradeToVersion($version);
        }
    }
    
    // Update version
    update_config_var('your_module_version', $toVersion);
    
    // Log upgrade
    logActivity("Upgraded your_module from {$fromVersion} to {$toVersion}");
}

/**
 * Execute upgrades for specific version
 * 
 * @param string $version Version to upgrade to
 */
function upgradeToVersion($version)
{
    switch ($version) {
        case '1.0.1':
            // Fix index typo from 1.0.0
            $sql = "ALTER TABLE `mod_your_table` 
                    DROP INDEX `idx_clinet_id`,
                    ADD INDEX `idx_client_id` (`client_id`)";
            full_query($sql);
            break;
            
        case '1.1.0':
            // Add new table for v1.1.0
            $sql = "CREATE TABLE IF NOT EXISTS `mod_your_cache` (
                `id` INT(10) AUTO_INCREMENT PRIMARY KEY,
                `key` VARCHAR(100) NOT NULL,
                `value` TEXT,
                `expires_at` DATETIME,
                UNIQUE KEY `idx_key` (`key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
            full_query($sql);
            break;
            
        case '1.1.1':
            // Add missing column
            $sql = "ALTER TABLE `mod_your_table` 
                    ADD COLUMN `metadata` TEXT AFTER `data`";
            full_query($sql);
            break;
            
        case '1.2.0':
            // Migrate data format
            migrateToNewDataFormat();
            break;
    }
}
```

## Changelog Management

### Changelog Structure

```php
<?php
/**
 * Module Changelog
 * 
 * Version history with detailed changes
 */

return [
    '1.2.0' => [
        'date'   => '2026-05-28',
        'type'   => 'feature',
        'changes' => [
            'NEW' => [
                'Added support for bulk operations',
                'Implemented new caching layer',
            ],
            'CHG' => [
                'Improved API response handling',
                'Updated database queries for better performance',
            ],
            'FIX' => [
                'Fixed timezone issue in scheduled tasks',
                'Corrected pagination in admin area',
            ],
        ],
    ],
    
    '1.1.1' => [
        'date'   => '2026-04-15',
        'type'   => 'fix',
        'changes' => [
            'FIX' => [
                'Fixed SQL error when client ID is null',
                'Corrected email notification format',
            ],
        ],
    ],
    
    '1.1.0' => [
        'date'   => '2026-03-01',
        'type'   => 'feature',
        'changes' => [
            'NEW' => [
                'Added client area widget',
                'Introduced webhook support',
            ],
            'CHG' => [
                'Refactored API client class',
            ],
        ],
    ],
    
    '1.0.0' => [
        'date'   => '2026-01-01',
        'type'   => 'initial',
        'changes' => [
            'NEW' => [
                'Initial release',
            ],
        ],
    ],
];
```

### Markdown Changelog Format

```markdown
# Changelog

All notable changes will be documented in this file.

## [1.2.0] - 2026-05-28

### Added
- Bulk operations support
- New caching layer for improved performance
- API rate limiting support

### Changed
- Improved API response handling
- Optimized database queries

### Fixed
- Timezone issue in scheduled tasks
- Pagination in admin area

## [1.1.1] - 2026-04-15

### Fixed
- SQL error when client ID is null
- Email notification format

## [1.1.0] - 2026-03-01

### Added
- Client area widget
- Webhook support

### Changed
- Refactored API client class

## [1.0.0] - 2026-01-01

### Added
- Initial release
```

## Version Compatibility

### WHMCS Version Compatibility

```php
/**
 * Check WHMCS version compatibility
 * 
 * @return array Compatibility info
 */
function checkWhmcsCompatibility()
{
    $minVersion = '8.0.0';
    $maxVersion = '8.9.9';
    $currentVersion = WHMCS_VERSION;
    
    $compatible = version_compare($currentVersion, $minVersion, '>=') &&
                  version_compare($currentVersion, $maxVersion, '<=');
    
    return [
        'compatible'    => $compatible,
        'min_required' => $minVersion,
        'max_supported' => $maxVersion,
        'current'       => $currentVersion,
    ];
}
```

### PHP Version Requirements

```php
/**
 * Check PHP version compatibility
 * 
 * @return array Compatibility info
 */
function checkPhpCompatibility()
{
    $minPhp = '7.4';
    $maxPhp = '8.3';
    $currentPhp = PHP_VERSION;
    
    $compatible = version_compare($currentPhp, $minPhp, '>=') &&
                  version_compare($currentPhp, $maxPhp, '<=');
    
    return [
        'compatible'    => $compatible,
        'min_required' => $minPhp,
        'max_supported' => $maxPhp,
        'current'       => $currentPhp,
    ];
}
```

## Deprecation Management

### Deprecation Notices

```php
/**
 * Mark function as deprecated
 * 
 * @param string $function Function name
 * @param string $version Version deprecated
 * @param string $replacement Replacement function
 */
function your_module_deprecate($function, $version, $replacement = null)
{
    $message = "Function {$function} is deprecated since version {$version}";
    
    if ($replacement) {
        $message .= ". Use {$replacement} instead.";
    }
    
    trigger_error($message, E_USER_DEPRECATED);
}

/**
 * Example usage of deprecation
 */
function old_function_name()
{
    your_module_deprecate(__FUNCTION__, '1.2.0', 'new_function_name');
    
    // Still call new function for backwards compatibility
    return new_function_name();
}
```

### Support Lifecycle

| Version | Release Date | End of Life | Status |
|---------|-------------|-------------|--------|
| 1.0.x | 2026-01-01 | 2026-12-31 | Security Fixes |
| 1.1.x | 2026-03-01 | 2027-03-01 | Security Fixes |
| 1.2.x | 2026-05-28 | Ongoing | Active Support |

---

## Related Skills and Workflows

- `module-upgrade-guide` - Upgrade procedures
- `module-release-checklist` - Release checklist
- `module-packaging-guide` - Package distribution
- `module-database-patterns` - Database migrations
