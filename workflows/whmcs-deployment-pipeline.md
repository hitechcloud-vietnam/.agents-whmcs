# WHMCS Deployment Pipeline Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Automate WHMCS module deployment pipeline from development to production.

## Prerequisites

- Git repository for version control
- Staging environment for testing
- Production environment access
- Deployment script/automation tool

## Deployment Pipeline Stages

### 1. Development Stage
```bash
# Local development
git checkout -b feature/new-module
# ... develop module ...
git commit -m "Add new module"
git push origin feature/new-module
```

### 2. Code Review Stage
```php
// Required checks before merge
- [ ] Code style follows WHMCS standards
- [ ] All functions documented
- [ ] No hardcoded credentials
- [ ] SQL injection prevention
- [ ] XSS protection implemented
- [ ] Unit tests pass
```

### 3. Staging Deployment
```bash
#!/bin/bash
# deploy-staging.sh
SSH_USER="staging"
SSH_HOST="staging.whmcs.example.com"
MODULE_NAME="mymodule"

# Upload module files
rsync -avz modules/servers/${MODULE_NAME}/ ${SSH_USER}@${SSH_HOST}:/var/www/whmcs/modules/servers/${MODULE_NAME}/

# Set permissions
ssh ${SSH_USER}@${SSH_HOST} "chown -R www-data:www-data /var/www/whmcs/modules/servers/${MODULE_NAME}/"

# Clear cache
ssh ${SSH_USER}@${SSH_HOST} "php /var/www/whmcs/admin/clearcache.php"

# Run automated tests
curl -X POST "https://staging.whmcs.example.com/api/test-module"
```

### 4. Automated Testing
```php
<?php
// Module validation test
class ModuleDeploymentTest {
    public function testModuleActivation(): void {
        $module = new MyModule();
        $result = $module->activate();

        $this->assertEquals('success', $result['status']);
        $this->assertTrue(Capsule::schema()->hasTable('mod_mymodule_data'));
    }

    public function testAccountCreation(): void {
        $params = [
            'domain' => 'test.example.com',
            'username' => 'testuser',
            'password' => 'SecurePass123!',
        ];

        $result = mymodule_CreateAccount($params);
        $this->assertEquals('success', $result);
    }

    public function testAccountTermination(): void {
        $result = mymodule_TerminateAccount(['serviceid' => 1]);
        $this->assertEquals('success', $result);
    }
}
```

### 5. Production Deployment
```bash
#!/bin/bash
# deploy-production.sh
set -e

MODULE_NAME="mymodule"
BACKUP_DIR="/backup/whmcs/modules/servers/${MODULE_NAME}-$(date +%Y%m%d)"

# Create backup
mkdir -p $BACKUP_DIR
cp -r /var/www/whmcs/modules/servers/${MODULE_NAME}/* $BACKUP_DIR/

# Download latest from repository
git clone git@github.com:company/whmcs-module.git /tmp/${MODULE_NAME}-new

# Deploy
rm -rf /var/www/whmcs/modules/servers/${MODULE_NAME}
cp -r /tmp/${MODULE_NAME}-new /var/www/whmcs/modules/servers/${MODULE_NAME}

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/servers/${MODULE_NAME}/

# Clear cache
php /var/www/whmcs/admin/clearcache.php

# Verify deployment
curl -s "https://whmcs.example.com/includes/modulehooks.php?module=${MODULE_NAME}&action=ping" | grep "ok"
```

## Rollback Procedure

```bash
#!/bin/bash
# rollback.sh
MODULE_NAME="mymodule"
BACKUP_DIR="/backup/whmcs/modules/servers"

# List available backups
ls -la $BACKUP_DIR | grep $MODULE_NAME

# Restore from backup
BACKUP_DATE="20240528"
rm -rf /var/www/whmcs/modules/servers/${MODULE_NAME}
cp -r $BACKUP_DIR/${MODULE_NAME}-${BACKUP_DATE} /var/www/whmcs/modules/servers/${MODULE_NAME}

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/servers/${MODULE_NAME}/

# Clear cache
php /var/www/whmcs/admin/clearcache.php
```

## CI/CD Integration

### GitHub Actions
```yaml
name: Deploy Module

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Run Tests
        run: |
          composer install
          ./vendor/bin/phpunit

      - name: Deploy to Staging
        if: github.ref == 'refs/heads/main'
        run: |
          ./deploy-staging.sh

      - name: Deploy to Production
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          ./deploy-production.sh
        env:
          SSH_KEY: ${{ secrets.SSH_KEY }}
```

## Health Checks

```php
<?php
// Post-deployment health check
add_hook('DailyCronJob', 1, function() {
    $module = 'mymodule';

    // Check module exists
    if (!is_dir(ROOTDIR . "/modules/servers/{$module}")) {
        logActivity("ERROR: Module {$module} not found");
        sendAlert("Module deployment failed");
        return;
    }

    // Check database tables
    $tables = ['mod_mymodule_data'];
    foreach ($tables as $table) {
        if (!Capsule::schema()->hasTable($table)) {
            logActivity("ERROR: Table {$table} missing");
        }
    }

    // Test API connectivity
    $api = new MyModuleAPI();
    if (!$api->testConnection()) {
        logActivity("ERROR: API connection failed");
    }
});
```

## Checklist

```
Pre-Deployment:
□ Version bumped (semantic versioning)
□ Changelog updated
□ All tests passing
□ Code reviewed and approved
□ Staging tested and approved

Deployment:
□ Backup created
□ Files uploaded
□ Permissions set
□ Cache cleared
□ Health check passed

Post-Deployment:
□ Monitor error logs
□ Check module activation
□ Test key functionality
□ Verify webhook delivery
□ Update deployment record
```

---

**Related Skills:**
- whmcs-deployment
- whmcs-testing-qa
- whmcs-ci-cd-workflow