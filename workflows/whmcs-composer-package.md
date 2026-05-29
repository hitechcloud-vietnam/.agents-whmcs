# WHMCS Composer Package Workflow

## Overview
This workflow guides you through creating and maintaining a Composer package for WHMCS modules.

## Prerequisites
- Composer installed
- Package structure ready

## Step-by-Step Guide

### Step 1: Create composer.json
```json
{
    "name": "vendor/whmcs-module",
    "description": "Description of your WHMCS module",
    "type": "whmcs-module",
    "license": "proprietary",
    "minimum-stability": "stable",
    "prefer-stable": true,
    "require": {
        "php": ">=8.0",
        "whmcs/whmcs": "^8.0",
        "guzzlehttp/guzzle": "^7.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^9.0",
        "mockery/mockery": "^1.0"
    },
    "autoload": {
        "psr-4": {
            "Vendor\\Module\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Vendor\\Module\\Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "post-install-cmd": [
            "php -r \"copy('src/config.php', 'config.php');\""
        ]
    },
    "extra": {
        "module": {
            "name": "Your Module",
            "version": "2.0.0",
            "position": "addon"
        }
    }
}
```

### Step 2: Install Dependencies
```bash
# Install production dependencies
composer install

# Install with dev dependencies
composer install --dev

# Update dependencies
composer update

# Update specific package
composer update vendor/package
```

### Step 3: Configure Autoloading
```php
// In your module bootstrap
require_once __DIR__ . '/vendor/autoload.php';

use Vendor\Module\Module;

$module = new Module();
$module->run();
```

## Composer Package Checklist

### Configuration
- [ ] composer.json valid
- [ ] Name follows convention
- [ ] Dependencies listed
- [ ] Autoloading configured

### Development
- [ ] Dependencies installable
- [ ] Tests runnable
- [ ] Scripts working
