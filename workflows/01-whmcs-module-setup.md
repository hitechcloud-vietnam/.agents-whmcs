# WHMCS Module Setup Workflow

## Overview
This workflow guides you through setting up a new WHMCS custom module from scratch, including directory structure, boilerplate code, and testing setup.

## Prerequisites
- WHMCS installation (v8.x or later)
- PHP 8.0+
- Composer installed
- Git repository initialized

## Step 1: Environment Preparation

```bash
# Clone or navigate to your WHMCS installation
cd /var/www/whmcs

# Set correct permissions
chmod 755 storage/ -R
chmod 755 modules/ -R
chmod 644 config.php
```

## Step 2: Module Directory Structure

Create the following structure for your module:

```
modules/
└── addons/
    └── your_module_name/
        ├── README.md
        ├── composer.json
        ├── CHANGELOG.md
        ├── your_module_name.php        # Main module file
        ├── src/
        │   ├── Controller/
        │   │   └── AdminController.php
        │   ├── Service/
        │   │   └── ModuleService.php
        │   └── Helper/
        │       └── Logger.php
        ├── templates/
        │   └── admin/
        │       └── overview.tpl
        ├── assets/
        │   ├── css/
        │   │   └── style.css
        │   └── js/
        │       └── module.js
        └── tests/
            ├── Unit/
            │   └── ModuleServiceTest.php
            └── Feature/
                └── IntegrationTest.php
```

## Step 3: Create Module Boilerplate

```php
<?php
// modules/addons/your_module_name/your_module_name.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Module\Addon\YourModule\Helper\Logger;

 /**
  * Define module information
  */
function your_module_name_config()
{
    return [
        'name'            => 'Your Module Name',
        'description'     => 'Description of what your module does',
        'author'          => 'Your Name',
        'language'        => 'english',
        'version'         => '1.0.0',
        'fields' => [
            'api_key' => [
                'Type'  => 'password',
                'Default' => '',
                'Description' => 'API Key for external service',
            ],
            'debug_mode' => [
                'Type'  => 'yesno',
                'Default' => false,
                'Description' => 'Enable debug logging',
            ],
        ],
    ];
}

/**
 * Activate module
 */
function your_module_name_activate()
{
    try {
        // Create database tables if needed
        $query = "CREATE TABLE IF NOT EXISTS `mod_your_module` (
            `id` INT NOT NULL AUTO_INCREMENT,
            `data` TEXT,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (`id`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";

        full_query($query);

        return [
            'status' => 'success',
            'description' => 'Module activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate: ' . $e->getMessage(),
        ];
    }
}

/**
 * Deactivate module
 */
function your_module_name_deactivate()
{
    try {
        // Clean up database tables
        $query = "DROP TABLE IF EXISTS `mod_your_module`";
        full_query($query);

        return [
            'status' => 'success',
            'description' => 'Module deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate: ' . $e->getMessage(),
        ];
    }
}

/**
 * Upgrade module
 */
function your_module_name_upgrade($vars)
{
    $currentVersion = $vars['version'];

    if ($currentVersion < '1.1.0') {
        // Migration to 1.1.0
        $query = "ALTER TABLE `mod_your_module` ADD COLUMN `new_field` VARCHAR(255)";
        full_query($query);
    }
}
```

## Step 4: Register Autoloader

```php
// modules/addons/your_module_name/composer.json
{
    "name": "vendor/your-module-name",
    "description": "Your WHMCS Module",
    "type": "project",
    "require": {
        "php": "^8.0",
        "whmcs/whmcs": "^8.0"
    },
    "autoload": {
        "psr-4": {
            "WHMCS\\Module\\Addon\\YourModule\\": "src/"
        }
    }
}
```

## Step 5: Initialize Git Repository

```bash
cd modules/addons/your_module_name
git init
git add .
git commit -m "Initial commit: WHMCS module boilerplate"
git tag -a v1.0.0 -m "Version 1.0.0"
```

## Verification Checklist

- [ ] Module directory structure created
- [ ] Main module file exists with required hooks
- [ ] Activation/deactivation functions implemented
- [ ] Database migrations prepared
- [ ] Composer.json autoloader configured
- [ ] Git repository initialized
- [ ] Initial commit created
- [ ] Module visible in WHMCS Admin > Extensions
- [ ] Module can be activated without errors
- [ ] Basic functionality tested

## Troubleshooting

### Common Issues

**Module not appearing in list:**
- Clear WHMCS cache: `php artisan cache:clear`
- Check file permissions: directories 755, files 644

**Activation fails:**
- Check PHP error logs
- Verify database credentials in config.php
- Ensure MySQL user has CREATE TABLE permissions

**Autoloader issues:**
- Run `composer dump-autoload`
- Check PSR-4 namespace matches directory structure
