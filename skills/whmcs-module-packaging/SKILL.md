# WHMCS Module Packaging Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for packaging WHMCS modules for distribution.

## When to Use

- Preparing module releases
- Creating installable packages
- Version management

## Packaging Patterns

### module.json Structure
```json
{
    "id": "whmcs_module",
    "name": "Module Name",
    "description": "Module description",
    "version": "1.0.0",
    "author": {
        "name": "Author Name",
        "email": "email@example.com"
    },
    "whmcs_version": "8.0",
    "php_version": "8.1",
    "dependencies": {
        "extensions": ["curl", "json"]
    }
}
```

### Release Script
```bash
#!/bin/bash
VERSION=${1:-"1.0.0"}
MODULE_NAME="modulename"

# Clean
rm -rf dist/
mkdir -p dist

# Create package structure
cp -r modules/$MODULE_NAME dist/
cp module.json dist/
cp README.md dist/
cp CHANGELOG.md dist/
cp LICENSE.md dist/

# Create archive
cd dist
zip -r ${MODULE_NAME}-v${VERSION}.zip .
cd ..

echo "Package created: dist/${MODULE_NAME}-v${VERSION}.zip"
```

### Installation Script
```php
<?php
// install.php - Module installation script
if (!defined("WHMCS")) { die("Direct access denied"); }

function install_{module}(): bool {
    try {
        // Create tables
        Capsule::schema()->create('mod_{module}_data', function($t) {
            $t->increments('id');
            $t->string('name');
            $t->timestamps();
        });

        // Set default settings
        Capsule::table('mod_{module}_settings')->insert([
            ['key' => 'version', 'value' => '1.0.0'],
            ['key' => 'enabled', 'value' => '1'],
        ]);

        logActivity('{Module} installed successfully');
        return true;

    } catch (\Exception $e) {
        logActivity('{Module} installation failed: ' . $e->getMessage());
        return false;
    }
}

function uninstall_{module}(): bool {
    try {
        Capsule::schema()->dropIfExists('mod_{module}_data');
        Capsule::schema()->dropIfExists('mod_{module}_settings');
        logActivity('{Module} uninstalled');
        return true;
    } catch (\Exception $e) {
        return false;
    }
}
```

---

**Related Skills:**
- whmcs-deployment
- whmcs-testing-qa
- whmcs-security-hardening
