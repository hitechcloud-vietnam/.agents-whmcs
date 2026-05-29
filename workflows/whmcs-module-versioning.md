# WHMCS Module Versioning Workflow

## Overview
This workflow guides you through managing versions for WHMCS modules.

## Prerequisites
- Git repository
- Semantic versioning knowledge
- Release process

## Step-by-Step Guide

### Step 1: Semantic Versioning
```
Version format: MAJOR.MINOR.PATCH
- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes (backward compatible)

Examples:
- 1.0.0 -> 1.0.1 (patch)
- 1.0.1 -> 1.1.0 (minor)
- 1.1.0 -> 2.0.0 (major)
```

### Step 2: Version File
```php
// version.php
<?php
return [
    'version' => '2.0.0',
    'release_date' => '2024-01-15',
    'minimum_whmcs_version' => '8.0.0',
    'changelog' => [
        '2.0.0' => [
            'date' => '2024-01-15',
            'changes' => [
                'Breaking: Removed legacy API support',
                'Added: New dashboard widget',
                'Fixed: Sync timeout issue',
            ],
        ],
        '1.5.2' => [
            'date' => '2024-01-10',
            'changes' => [
                'Fixed: Error in webhook handler',
            ],
        ],
    ],
];
```

### Step 3: Version Check in Module
```php
// yourmodule.php
<?php
function yourmodule_output()
{
    $version = \WHMCS\Module\YourModule\Version::get();
    
    // Check minimum WHMCS version
    $minWhmcs = $version->getMinimumWhmcsVersion();
    if (version_compare(\App::getVersion(), $minWhmcs, '<')) {
        return '<div class="alert alert-danger">
            This module requires WHMCS ' . $minWhmcs . ' or higher.
        </div>';
    }
}
```

### Step 4: Version Commands
```bash
# Current version
git describe --tags

# List tags
git tag -l

# Create new version tag
git tag -a v2.0.0 -m "Release version 2.0.0"

# Push tags
git push origin --tags
```

## Versioning Checklist

### Version Management
- [ ] Semantic versioning followed
- [ ] Version file updated
- [ ] Changelog maintained
- [ ] Tags created

### Release
- [ ] Version bumped correctly
- [ ] Release notes written
- [ ] Artifacts built
- [ ] Deployed
