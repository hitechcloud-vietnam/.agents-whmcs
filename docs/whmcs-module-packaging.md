# WHMCS Module Packaging

Complete guide for packaging WHMCS modules for distribution.

## Overview

Proper packaging ensures easy installation and updates for end users.

## Package Structure

### Standard Module Structure

```
modules/
├── servers/
│   └── yourmodule/
│       ├── yourmodule.php          # Main module file
│       ├── lib/
│       │   ├── ApiClient.php
│       │   ├── Exception.php
│       │   └── helpers.php
│       ├── templates/
│       │   └── clientarea.tpl
│       ├── views/
│       │   └── dashboard.phtml
│       └── migrations/
│           └── create_tables.php
├── addons/
│   └── youraddon/
│       └── youraddon.php
└──registrars/
    └── yourregistrar/
        └── yourregistrar.php
```

### Addon Module Structure

```
modules/
└── addons/
    └── youraddon/
        ├── youraddon.php
        ├── admin.php
        ├── client.php
        ├── hooks.php
        ├── Api/
        │   └── Client.php
        ├── templates/
        │   ├── admin.tpl
        │   └── client.tpl
        ├── views/
        │   ├── settings.phtml
        │   └── dashboard.phtml
        └── assets/
            ├── css/
            │   └── style.css
            └── js/
                └── script.js
```

## Composer Configuration

### Package composer.json

```json
{
    "name": "vendor/whmcs-module",
    "description": "WHMCS provisioning module",
    "type": "project",
    "license": "proprietary",
    "require": {
        "php": ">=7.4",
        "guzzlehttp/guzzle": "^7.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^9.0"
    },
    "autoload": {
        "psr-4": {
            "Vendor\\YourModule\\": "lib/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "post-install-cmd": [
            "php -r \"echo 'Module installed successfully';\""
        ]
    }
}
```

## Release Package

### Creating Release Package

```php
<?php
/**
 * Create distribution package
 */
class ModulePackager
{
    private string $modulePath;
    private string $outputPath;
    
    public function __construct(string $modulePath, string $outputPath)
    {
        $this->modulePath = $modulePath;
        $this->outputPath = $outputPath;
    }
    
    public function createPackage(string $version): string
    {
        $packageName = "yourmodule-{$version}.zip";
        $packagePath = "{$this->outputPath}/{$packageName}";
        
        $zip = new ZipArchive();
        $zip->open($packagePath, ZipArchive::CREATE | ZipArchive::OVERWRITE);
        
        $this->addDirectoryToZip($this->modulePath, $zip, '');
        
        // Add version file
        $zip->addFromString('VERSION', $version);
        $zip->addFromString('MANIFEST.json', json_encode([
            'name' => 'yourmodule',
            'version' => $version,
            'release_date' => date('Y-m-d'),
            'checksum' => hash_file('sha256', $packagePath),
        ], JSON_PRETTY_PRINT));
        
        $zip->close();
        
        return $packagePath;
    }
    
    private function addDirectoryToZip(string $dir, ZipArchive $zip, string $basePath): void
    {
        $files = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($dir)
        );
        
        foreach ($files as $file) {
            if ($file->isDir()) continue;
            
            $filePath = $file->getRealPath();
            $relativePath = $basePath . basename($dir) . '/' . 
                substr($filePath, strlen($this->modulePath) + 1);
            
            $zip->addFile($filePath, $relativePath);
        }
    }
}
```

### Package Manifest

```json
{
    "name": "yourmodule",
    "version": "1.5.2",
    "type": "server",
    "category": "Provisioning",
    "author": {
        "name": "Your Company",
        "email": "support@example.com",
        "website": "https://example.com"
    },
    "description": "Full-featured provisioning module",
    "whmcs_version": "8.0+",
    "release_date": "2024-01-15",
    "changes": [
        "Added new feature",
        "Fixed critical bug",
        "Improved performance"
    ],
    "requirements": {
        "php": "7.4+",
        "extensions": ["curl", "json"]
    },
    "files": [
        "modules/servers/yourmodule/yourmodule.php",
        "modules/servers/yourmodule/lib/ApiClient.php",
        "modules/servers/yourmodule/lib/Exception.php"
    ],
    "hooks": [
        "ClientAreaPrimaryNavbar",
        "ServiceProvision"
    ],
    "database_tables": [
        "mod_yourmodule_data"
    ],
    "configuration": {
        "required": ["api_key", "server_url"],
        "optional": ["timeout", "debug_mode"]
    }
}
```

## Installation Script

### Automated Installation

```php
<?php
/**
 * Module installation script
 */
function yourmodule_install(): bool
{
    try {
        // Create database tables
        if (!Capsule::schema()->hasTable('mod_yourmodule_data')) {
            Capsule::schema()->create('mod_yourmodule_data', function($table) {
                $table->increments('id');
                $table->integer('service_id')->unsigned();
                $table->string('account_id', 100);
                $table->string('status', 50)->default('pending');
                $table->timestamps();
                
                $table->index(['service_id']);
                $table->index(['account_id']);
            });
        }
        
        // Set default configuration
        Capsule::table('tblconfiguration')->updateOrInsert(
            ['setting' => 'YourModuleDefaultSetting'],
            ['value' => 'default_value']
        );
        
        // Register hooks
        add_hook('ClientAreaPrimaryNavbar', 1, 'yourmodule_clientNavbarHook');
        
        return true;
        
    } catch (Exception $e) {
        logModuleCall('yourmodule', 'install', [], [], 'error: ' . $e->getMessage());
        return false;
    }
}
```

### Upgrade Script

```php
<?php
/**
 * Module upgrade script
 */
function yourmodule_upgrade(string $fromVersion, string $toVersion): bool
{
    try {
        logActivity("Upgrading YourModule from {$fromVersion} to {$toVersion}");
        
        // Version 1.0 -> 1.1
        if (version_compare($fromVersion, '1.1', '<')) {
            upgradeToV1_1();
        }
        
        // Version 1.1 -> 1.2
        if (version_compare($fromVersion, '1.2', '<')) {
            upgradeToV1_2();
        }
        
        // Update version in database
        Capsule::table('tblconfiguration')
            ->updateOrInsert(
                ['setting' => 'YourModuleVersion'],
                ['value' => $toVersion]
            );
        
        return true;
        
    } catch (Exception $e) {
        logModuleCall('yourmodule', 'upgrade', [
            'from' => $fromVersion,
            'to' => $toVersion
        ], [], 'error: ' . $e->getMessage());
        return false;
    }
}

function upgradeToV1_1(): void
{
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'api_response')) {
        Capsule::schema()->table('mod_yourmodule_data', function($table) {
            $table->text('api_response')->nullable();
        });
    }
    
    logActivity('YourModule: Applied upgrade to v1.1');
}

function upgradeToV1_2(): void
{
    Capsule::schema()->create('mod_yourmodule_settings', function($table) {
        $table->increments('id');
        $table->string('setting_key', 100)->unique();
        $table->text('setting_value');
        $table->timestamps();
    });
    
    logActivity('YourModule: Applied upgrade to v1.2');
}
```

## Distribution

### Creating Distribution Zip

```bash
#!/bin/bash
# build-package.sh

MODULE_VERSION=$(cat VERSION)
OUTPUT_FILE="yourmodule-${MODULE_VERSION}.zip"

# Create temporary directory
TEMP_DIR=$(mktemp -d)

# Copy module files
cp -r modules/servers/yourmodule "$TEMP_DIR/"
cp composer.json "$TEMP_DIR/"
cp composer.lock "$TEMP_DIR/"
cp README.md "$TEMP_DIR/"
cp LICENSE "$TEMP_DIR/"
cp CHANGELOG.md "$TEMP_DIR/"

# Create zip
cd "$TEMP_DIR"
zip -r "$OLDPWD/$OUTPUT_FILE" .
cd "$OLDPWD"

# Clean up
rm -rf "$TEMP_DIR"

echo "Created: $OUTPUT_FILE"
echo "SHA256: $(sha256sum $OUTPUT_FILE)"
```

## Version Tagging

### Git Tagging

```bash
# Create version tag
git tag -a v1.5.0 -m "Release version 1.5.0

Features:
- New API endpoint support
- Improved error handling
- Performance optimizations

Breaking changes:
- None"

# Push tag
git push origin v1.5.0

# Create release
gh release create v1.5.0 \
    --title "Version 1.5.0" \
    --notes "See CHANGELOG.md for details"
```

## Best Practices

1. **Include README** - Clear installation and configuration instructions
2. **Version properly** - Follow semantic versioning
3. **Test thoroughly** - Verify installation works on clean WHMCS
4. **Document changes** - Keep detailed changelog
5. **Sign releases** - Use GPG for security
6. **Provide checksums** - Help users verify integrity

## Related Documentation

- [whmcs-module-lifecycle.md](whmcs-module-lifecycle.md)
- [whmcs-module-versioning.md](whmcs-module-versioning.md)
