# WHMCS Deployment & Release Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for deploying WHMCS modules to production and managing releases.

## When to Use

- Releasing a new module
- Updating an existing module
- Packaging modules for distribution

## Deployment Steps

### 1. Pre-Release Checklist

```bash
# Code Review
- [ ] All functions implemented
- [ ] No TODO or FIXME comments
- [ ] Error handling complete
- [ ] Logging appropriate

# Security Review
- [ ] No hardcoded credentials
- [ ] CSRF protection on all forms
- [ ] Input validation complete
- [ ] Output escaping in templates

# Testing
- [ ] Unit tests passing
- [ ] Integration tests passing
- [ ] Manual testing complete
- [ ] Edge cases handled
```

### 2. Version Bumping

```php
// In module file
define('{MODULE}_VERSION', '1.0.0');

// In config function
'version' => [
    'FriendlyName' => 'Version',
    'Type' => 'System',
    'Value' => '1.0.0',
],

// In whmcs.json (if applicable)
{
    "version": "1.0.0",
    "release_date": "2026-05-28"
}
```

### 3. Module Packaging

```bash
# Create release directory
mkdir -p release/{module}/modules

# Copy module files
cp -r modules/servers/{module} release/{module}/modules/servers/
cp -r modules/gateways/{module} release/{module}/modules/gateways/
cp README.md release/{module}/

# Create ZIP archive
cd release
zip -r {module}-v1.0.0.zip {module}/
```

### 4. Installation Guide

```markdown
## Installation

1. Download the module package
2. Extract to a temporary folder
3. Upload the `modules` folder to your WHMCS root
4. Set correct permissions (755 for PHP files, 644 for config)
5. Go to WHMCS Admin > Setup > Servers/Products
6. Configure the module with your API credentials
7. Test the connection

## Upgrading

1. Backup your current configuration
2. Deactivate the existing module
3. Upload new module files
4. Reactivate and configure
```

## Release Checklist

### Module Files
- [ ] Main module file with version
- [ ] Logo (80x80px PNG)
- [ ] hooks.php if applicable
- [ ] lang/english.php translations
- [ ] lib/ API client files
- [ ] templates/ Smarty templates

### Documentation
- [ ] README.md with installation
- [ ] CHANGELOG.md with changes
- [ ] Configuration guide
- [ ] Troubleshooting section

### Testing
- [ ] Test on staging environment
- [ ] Verify all module functions
- [ ] Test upgrade path
- [ ] Test backup/restore

## Installation Methods

### Manual Upload
```bash
# Upload via FTP/SFTP
/modules/servers/{module}/
/modules/gateways/{module}.php
/modules/addons/{module}/
```

### Deployment Script
```php
<?php
// deploy.php - Run once to install
define('WHMCS_ROOT', '/path/to/whmcs/');

require_once WHMCS_ROOT . '/includes/init.php';

$moduleFiles = [
    'modules/servers/providername/providername.php',
    'modules/servers/providername/lib/ApiClient.php',
];

foreach ($moduleFiles as $file) {
    $source = __DIR__ . '/source/' . $file;
    $dest = WHMCS_ROOT . $file;

    if (file_exists($source)) {
        copy($source, $dest);
        echo "Installed: $file\n";
    }
}

echo "Module deployment complete\n";
```

## Rollback Procedure

```php
// rollback.php
function rollbackModule(string $module, string $version): void {
    $backupDir = '/backups/modules/' . $module . '/' . $version;

    // Restore files
    $files = glob($backupDir . '/*');
    foreach ($files as $file) {
        $relPath = str_replace($backupDir . '/', '', $file);
        copy($file, WHMCS_ROOT . $relPath);
    }

    // Notify
    logActivity('Module rollback: ' . $module . ' to v' . $version);
}
```

## Post-Deployment

- [ ] Verify module appears in WHMCS
- [ ] Test configuration save
- [ ] Test module functionality
- [ ] Monitor error logs
- [ ] Document known issues

---

**Related Skills:**
- whmcs-testing-qa
- whmcs-security-hardening
- whmcs-module-bundling