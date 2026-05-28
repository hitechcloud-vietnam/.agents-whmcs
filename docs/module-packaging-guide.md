# WHMCS Module Packaging Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers packaging WHMCS modules for distribution, including module structure, archive creation, manifest files, and submission to the WHMCS Marketplace.

---

## Package Structure

### Standard Module Package

```
your_module_v1.2.0/
  ├── module.xml              # WHMCS module manifest
  ├── CHANGELOG.md            # Version history
  ├── README.md              # Installation instructions
  ├── LICENSE                # License file
  │
  └── modules/
      ├── addons/
      │   └── your_addon/
      │       ├── your_addon.php
      │       ├── templates/
      │       └── assets/
      │
      ├── gateways/
      │   └── your_gateway/
      │       ├── your_gateway.php
      │       ├── callback.php
      │       └── logo.png
      │
      ├── registrars/
      │   └── your_registrar/
      │       └── your_registrar.php
      │
      └── notifications/
          └── your_notification/
              └── your_notification.php
```

## Module Manifest

### module.xml Format

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module>
    <name>Your Module Name</name>
    <version>1.2.0</version>
    <description>Module description</description>
    <author>Your Name</author>
    <authorUrl>https://example.com</authorUrl>
    <authorEmail>support@example.com</authorEmail>
    
    <license>proprietary</license>
    
    <requires>
        <php version="7.4"/>
        <whmcs version="8.0" min="8.0.0" max="8.9.9"/>
    </requires>
    
    <types>
        <type>addon</type>
        <!-- or gateway, registrar, notification -->
    </types>
    
    <files>
        <file>modules/addons/your_addon/your_addon.php</file>
        <file>modules/addons/your_addon/templates/</file>
        <file>modules/addons/your_addon/assets/</file>
    </files>
    
    <hooks>
        <hook>ClientAdd</hook>
        <hook>ClientEdit</hook>
    </hooks>
    
    <options>
        <option name="auto_activate" type="boolean">true</option>
    </options>
</module>
```

### Combined Module Manifest

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module>
    <name>Your Module Suite</name>
    <version>1.2.0</version>
    <description>Complete module suite including gateway, addon, and registrar</description>
    <author>Your Name</author>
    
    <modules>
        <module>
            <name>gw_your_gateway</name>
            <type>gateway</type>
            <version>1.2.0</version>
            <files>
                <file>modules/gateways/your_gateway/</file>
            </files>
        </module>
        
        <module>
            <name>your_addon</name>
            <type>addon</type>
            <version>1.2.0</version>
            <files>
                <file>modules/addons/your_addon/</file>
            </files>
            <hooks>
                <hook>ClientAdd</hook>
            </hooks>
        </module>
    </modules>
</module>
```

## README.md Template

```markdown
# Your Module Name

Brief description of what your module does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Requirements

- WHMCS 8.0 or higher
- PHP 7.4 or higher
- External API account (if applicable)

## Installation

1. Download the module package
2. Extract the archive
3. Upload the `modules` folder to your WHMCS root
4. Navigate to Setup > Addon Modules (or similar)
5. Activate and configure the module

## Configuration

| Setting | Description | Default |
|---------|-------------|---------|
| API Key | Your API key | - |
| Test Mode | Enable test mode | No |
| Debug | Enable debug logging | No |

## Usage

Describe how to use the module after installation.

## Troubleshooting

### Issue 1
Solution description.

### Issue 2
Solution description.

## Support

For support, contact:
- Email: support@example.com
- Website: https://example.com/support

## License

Proprietary - All rights reserved
```

## Archive Creation

### Manual Archive Creation

```bash
# Create module package
cd your_module_v1.2.0
zip -r your_module_v1.2.0.zip .
```

### PowerShell Script

```powershell
# Create module package
$version = "1.2.0"
$moduleName = "your_module"
$archiveName = "${moduleName}_v${version}.zip"

# Create archive
Compress-Archive -Path ".\modules\*" -DestinationPath $archiveName -Force

# Add documentation
Compress-Archive -Path ".\README.md" -DestinationPath $archiveName -Update
Compress-Archive -Path ".\CHANGELOG.md" -DestinationPath $archiveName -Update
Compress-Archive -Path ".\LICENSE" -DestinationPath $archiveName -Update
Compress-Archive -Path ".\module.xml" -DestinationPath $archiveName -Update

Write-Host "Created: $archiveName"
```

## Directory Structure Guidelines

### Files to Include

```
your_module/
├── modules/                  # Required: WHMCS modules folder
│   ├── addons/
│   │   └── your_addon/
│   ├── gateways/
│   │   └── your_gateway/
│   └── registrars/
│       └── your_registrar/
├── hooks/                    # Optional: Custom hook files
├── templates/                # Optional: Custom templates
├── translations/            # Optional: Language files
├── CHANGELOG.md              # Required: Version history
├── README.md                 # Required: Installation guide
├── LICENSE                   # Required: License file
└── module.xml               # Required: Module manifest
```

### Files to Exclude

```
your_module/
├── .git/                     # Version control
├── .gitignore
├── node_modules/             # Build dependencies
├── tests/                    # Test files
├── dist/                     # Build output
├── composer.json             # PHP dependencies
├── package.json              # Build dependencies
├── *.psd                    # Design files
├── *.ai                     # Design files
├── .DS_Store                 # System files
└── Thumbs.db                 # System files
```

## Package Validation

### Pre-Submission Checklist

- [ ] All source files present
- [ ] module.xml valid and complete
- [ ] README.md clear and accurate
- [ ] CHANGELOG.md updated
- [ ] LICENSE file included
- [ ] Module activates without errors
- [ ] No debug or test code left in production
- [ ] No sensitive data (API keys, etc.) included
- [ ] Version number correct
- [ ] Works on fresh WHMCS installation

### Validation Script

```bash
#!/bin/bash
# Validate module package

echo "Validating module package..."

# Check for required files
required_files=(
    "modules/addons/your_addon/your_addon.php"
    "module.xml"
    "README.md"
    "CHANGELOG.md"
    "LICENSE"
)

for file in "${required_files[@]}"; do
    if [ -f "$file" ]; then
        echo "✓ $file exists"
    else
        echo "✗ $file missing"
        exit 1
    fi
done

# Check module.xml validity
if xmllint --noout module.xml 2>/dev/null; then
    echo "✓ module.xml is valid XML"
else
    echo "✗ module.xml has errors"
    exit 1
fi

# Check PHP syntax
echo "Checking PHP syntax..."
find modules -name "*.php" -exec php -l {} \; \;

echo "Validation complete!"
```

---

## Related Skills and Workflows

- `module-submission-marketplace` - Marketplace submission process
- `module-release-checklist` - Pre-release checklist
- `module-versioning-guide` - Version management
- `module-testing-strategies` - Testing before packaging
