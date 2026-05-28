# WHMCS Deployment Best Practices Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Best practices for deploying WHMCS modules.

## Pre-Deployment Checklist

```
Development:
□ Code follows WHMCS standards
□ All functions named correctly
□ Error handling in place
□ CSRF protection on forms
□ Input validation done
□ SQL injection prevention
□ XSS prevention
□ No hardcoded credentials
□ Logs without sensitive data

Testing:
□ Module activates without errors
□ Module deactivates without errors
□ No PHP errors/warnings
□ All features work as expected
□ Tested on fresh WHMCS install
□ Tested with upgrade from previous version

Documentation:
□ README.md created
□ Installation instructions included
□ Configuration guide ready
□ Changelog updated
□ version number bumped
```

## Deployment Steps

### 1. Code Freeze
```bash
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

### 2. Package Creation
```bash
mkdir -p release
cp -r modules/{module}/ release/
cp README.md release/
cp CHANGELOG.md release/
zip -r {module}-v1.0.0.zip release/
```

### 3. Staging Deployment
```bash
# Deploy to staging
rsync -avz modules/{module}/ user@staging:/var/www/html/modules/addons/

# Or via Composer
composer require vendor/{module} --dev
```

### 4. Production Deployment
```bash
# Backup current version
cp -r /var/www/html/modules/addons/{module} /backup/{module}-$(date +%Y%m%d)

# Deploy new version
cp -r staging/modules/{module} /var/www/html/modules/addons/

# Set permissions
chown -R www-data:www-data /var/www/html/modules/addons/{module}
chmod -R 755 /var/www/html/modules/addons/{module}
chmod -R 775 /var/www/html/modules/addons/{module}/templates_c
```

### 5. Post-Deployment Verification
```bash
# Check WHMCS logs
tail -f /var/www/html/whmcs/logs/module.log

# Test module activation
# Test basic functionality
# Monitor error logs
```

## Rollback Procedure

```bash
# Stop deployment
# Restore backup
cp -r /backup/{module}-20240101 /var/www/html/modules/addons/

# Clear any failed migrations
# Verify system state
```

## Output

Complete deployment package:
- Versioned release
- Installation guide
- Rollback procedure
- Verification checklist
