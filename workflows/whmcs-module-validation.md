# WHMCS Module Validation Workflow

## Description
Validate WHMCS modules before distribution.

## Validation Checklist

### Code Quality
- [ ] No syntax errors
- [ ] Proper error handling
- [ ] Input validation
- [ ] SQL injection prevention
- [ ] XSS prevention
- [ ] CSRF protection

### Security
- [ ] No hardcoded credentials
- [ ] Secure API calls
- [ ] Proper file permissions
- [ ] Callback IP validation

### WHMCS Compliance
- [ ] Follows module structure
- [ ] Proper return values
- [ ] Uses WHMCS APIs correctly
- [ ] Hooks properly implemented

## Steps

### Step 1: Syntax Check
```bash
# Check all PHP files
find . -name "*.php" -exec php -l {} \; 2>&1 | grep -v "No syntax errors"
```

### Step 2: Security Scan
```php
<?php
/**
 * Security validation script
 */

function validateModuleSecurity($modulePath)
{
    $issues = [];
    
    // Check for hardcoded passwords
    $files = glob("$modulePath/**/*.php");
    foreach ($files as $file) {
        $content = file_get_contents($file);
        
        if (preg_match('/password["\']?\s*[:=]\s*["\'][^"\']{8,}/i', $content)) {
            $issues[] = "Potential hardcoded password in $file";
        }
        
        if (preg_match('/api[_-]?key["\']?\s*[:=]\s*["\'][A-Za-z0-9]{20,}/i', $content)) {
            $issues[] = "Potential hardcoded API key in $file";
        }
    }
    
    // Check for SQL injection vulnerabilities
    // Check for XSS vulnerabilities
    
    return $issues;
}
```

### Step 3: Module Structure Validation
```php
<?php
/**
 * Validate module structure
 */

function validateModuleStructure($modulePath)
{
    $required = [
        'module.php',
        'LICENSE',
        'README.md',
    ];
    
    $missing = [];
    foreach ($required as $file) {
        if (!file_exists("$modulePath/$file")) {
            $missing[] = $file;
        }
    }
    
    return [
        'valid' => empty($missing),
        'missing' => $missing,
    ];
}
```

### Step 4: Functional Testing
```bash
# Test module activation
curl -X POST "https://whmcs.test/admin/module.php?module=clicodes_example&action=activate"

# Test with WHMCS CLI
php whmcscli modules test clicodes_example
```

### Step 5: Automated Validation Script
```bash
#!/bin/bash
# validate.sh

echo "Validating module..."

# Syntax check
echo "Checking syntax..."
find . -name "*.php" -exec php -l {} \; | grep -v "No syntax errors"

# Security scan
echo "Running security scan..."
php validate/security.php

# Structure check
echo "Checking structure..."
php validate/structure.php

echo "Validation complete."
```

## Validation Tools
| Tool | Purpose |
|------|---------|
| PHP CodeSniffer | Code style |
| PHPStan | Static analysis |
| SonarQube | Security scanning |
| Custom scripts | WHMCS specifics |

## Tags
- validation
- quality
- security
- testing