# WHMCS Module Packaging Workflow

## Description
Package WHMCS modules for distribution with proper structure and build process.

## Prerequisites
- Completed module development
- Git repository
- Composer (if dependencies)

## Steps

### Step 1: Define Package Structure
```
{module-name}/
├── module.php                    # Main module file
├── callback.php                  # Callback handler (if gateway)
├── LICENSE
├── README.md
├── CHANGELOG.md
├── composer.json                 # If using dependencies
├── build/
│   └── build.php                 # Build script
├── src/
│   └── ...                       # Source code
├── templates/
│   ├── admin/
│   └── client/
├── languages/
│   ├── english.php
│   └── ...                       # Other languages
└── tests/
    └── ...
```

### Step 2: Create Composer.json
```json
{
    "name": "vendor/whmcs-module-name",
    "description": "WHMCS Module Description",
    "type": "whmcs-module",
    "license": "proprietary",
    "require": {
        "php": "^8.1"
    },
    "autoload": {
        "psr-4": {
            "Vendor\\Module\\": "src/"
        }
    },
    "extra": {
        "whmcs-module": {
            "type": "addon|gateway|registrar|server|notification"
        }
    }
}
```

### Step 3: Create Build Script
```php
#!/usr/bin/env php
<?php
/**
 * WHMCS Module Build Script
 */

$moduleName = 'clicodes_example';
$version = trim(file_get_contents(__DIR__ . '/../VERSION'));
$buildDir = __DIR__ . '/build';
$distDir = __DIR__ . '/../dist';

// Clean dist directory
if (is_dir($distDir)) {
    exec("rm -rf $distDir");
}
mkdir($distDir);

// Create archive name
$archiveName = "{$moduleName}-v{$version}.zip";

// Build process
echo "Building {$moduleName} v{$version}...\n";

// 1. Run tests
echo "Running tests...\n";
exec(__DIR__ . '/run-tests.sh');

// 2. Minify files (optional)
echo "Optimizing files...\n";

// 3. Create distribution
echo "Creating distribution...\n";
$zip = new ZipArchive();
$zip->open($distDir . '/' . $archiveName, ZipArchive::CREATE);

// Add all files recursively
function addFilesToZip($zip, $dir, $basePath) {
    $files = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator($dir),
        RecursiveIteratorIterator::SELF_FIRST
    );
    
    foreach ($files as $file) {
        if ($file->isDir()) {
            continue;
        }
        
        $relativePath = str_replace($basePath . '/', '', $file->getPathname());
        
        // Skip test files and development files
        if (strpos($relativePath, 'tests/') === 0 ||
            strpos($relativePath, '.git/') === 0 ||
            strpos($file->getFilename(), '.map') !== false) {
            continue;
        }
        
        $zip->addFile($file->getPathname(), $relativePath);
    }
}

addFilesToZip($zip, __DIR__ . '/../', __DIR__ . '/..');
$zip->close();

// Create checksum
$checksum = hash_file('sha256', $distDir . '/' . $archiveName);
file_put_contents($distDir . '/' . $archiveName . '.sha256', $checksum);

echo "Build complete: {$archiveName}\n";
echo "Checksum: {$checksum}\n";
```

### Step 4: Create Release Metadata
```json
{
    "name": "clicodes_example",
    "version": "1.0.0",
    "whmcs_version": "^8.0",
    "php_version": "^8.1",
    "release_date": "2024-01-15",
    "changelog": [
        "Initial release",
        "Bug fixes"
    ],
    "requirements": {
        "extensions": ["curl", "json"],
        "memory_limit": "256M"
    }
}
```

### Step 5: Create Installation Script
```php
<?php
/**
 * WHMCS Module Installation Script
 * Run before copying files
 */

$requirements = [
    'php_version' => '8.1',
    'whmcs_version' => '8.0',
    'extensions' => ['curl', 'json', 'pdo_mysql'],
];

function checkRequirements($requirements) {
    $errors = [];
    
    // Check PHP version
    if (version_compare(PHP_VERSION, $requirements['php_version'], '<')) {
        $errors[] = "PHP {$requirements['php_version']}+ required. Current: " . PHP_VERSION;
    }
    
    // Check extensions
    foreach ($requirements['extensions'] ?? [] as $ext) {
        if (!extension_loaded($ext)) {
            $errors[] = "Required extension: {$ext}";
        }
    }
    
    // Check WHMCS version
    $whmcsVersion = defined('WHMCS_VERSION') ? WHMCS_VERSION : '0';
    if (version_compare($whmcsVersion, $requirements['whmcs_version'], '<')) {
        $errors[] = "WHMCS {$requirements['whmcs_version']}+ required. Current: {$whmcsVersion}";
    }
    
    return $errors;
}

$errors = checkRequirements($requirements);

if (!empty($errors)) {
    echo "Requirements not met:\n";
    foreach ($errors as $error) {
        echo "- $error\n";
    }
    exit(1);
}

echo "All requirements met.\n";
```

### Step 6: Create Distribution Archive
```bash
#!/bin/bash
# build.sh

cd /path/to/module
VERSION=$(cat VERSION)

# Update version in module file
sed -i "s/version' => '[0-9.]*'/version' => '$VERSION'/" module.php

# Build
php build/build.php

# Create GitHub release
if command -v gh &> /dev/null; then
    gh release create "v$VERSION" \
        --title "Release v$VERSION" \
        --notes-file CHANGELOG.md \
        "dist/${module}-v${VERSION}.zip"
fi
```

### Step 7: Sign Release
```bash
# Create GPG signature
gpg --armor --detach-sign dist/module-v1.0.0.zip

# This creates: dist/module-v1.0.0.zip.asc
```

## Package Contents Checklist
- [ ] Main module file
- [ ] Configuration/License
- [ ] README with installation instructions
- [ ] CHANGELOG
- [ ] Version file
- [ ] Language files
- [ ] Templates (if any)
- [ ] Installation requirements

## Distribution Formats
1. ZIP archive (most common)
2. Composer package
3. GitHub release
4. WHMCS Marketplace upload

## Tags
- packaging
- distribution
- release
- build