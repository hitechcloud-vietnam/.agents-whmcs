# WHMCS Plugin Repository Setup Workflow

## Overview
This workflow guides you through setting up a plugin repository for WHMCS modules.

## Prerequisites
- Git repository hosting (GitHub/GitLab)
- Release hosting

## Step-by-Step Guide

### Step 1: Repository Structure
```bash
yourmodule/
├── .github/
│   └── workflows/
│       └── release.yml
├── src/
│   └── yourmodule/
│       ├── yourmodule.php
│       ├── lib/
│       └── templates/
├── tests/
├── docs/
├── composer.json
├── README.md
├── CHANGELOG.md
└── LICENSE
```

### Step 2: Release Workflow
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Install dependencies
        run: composer install --no-dev

      - name: Run tests
        run: ./vendor/bin/phpunit

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*.zip
          generate_release_notes: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Step 3: Composer Repository
```json
// composer.json for the module
{
    "name": "vendor/yourmodule",
    "description": "WHMCS Module Description",
    "type": "whmcs-module",
    "license": "proprietary",
    "require": {
        "php": ">=8.0",
        "whmcs/whmcs": "^8.0"
    },
    "autoload": {
        "psr-4": {
            "Vendor\\YourModule\\": "src/"
        }
    }
}
```

### Step 4: Set Up Private Repository
```json
// For private repository
{
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/vendor/yourmodule"
        }
    ],
    "require": {
        "vendor/yourmodule": "^2.0"
    }
}
```

## Repository Setup Checklist

### Structure
- [ ] Standard directory layout
- [ ] CI/CD configured
- [ ] Documentation included

### Release
- [ ] Tags working
- [ ] GitHub releases working
- [ ] Packages downloadable

### Composer
- [ ] composer.json valid
- [ ] Autoloading works
- [ ] Dependencies listed
