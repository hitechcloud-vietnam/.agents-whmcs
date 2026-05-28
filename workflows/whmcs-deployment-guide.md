# WHMCS Module Deployment Guide

## Purpose

Complete guide to deploying WHMCS modules with best practices. Covers version management, packaging, testing, deployment procedures, rollback strategies, and post-deployment verification.

## Prerequisites

- WHMCS 7.0+ installation
- Module source code ready for deployment
- Access to development, staging, and production environments
- Deployment tools (git, composer, rsync)
- Release management process

## Workflow Steps

### Step 1: Pre-Deployment Preparation

```
Deployment Readiness Checklist:

Code Quality:
□ All tests passing
□ Code review completed
□ Security scan passed
□ No hardcoded credentials
□ Changelog updated
□ Version bumped

Documentation:
□ README.md updated
□ Installation guide complete
□ Configuration documentation
□ Migration guide (if needed)
□ API documentation (if applicable)

Testing:
□ Unit tests passing
□ Integration tests passing
□ Staging deployment verified
□ Performance tested
□ Security reviewed

Release:
□ Version tag created
□ Release branch merged
□ Deployment package ready
□ Rollback plan documented
```

### Step 2: Version Management

```bash
# ===========================================
# Semantic Versioning
# ===========================================

# Version format: MAJOR.MINOR.PATCH
# MAJOR - Breaking changes
# MINOR - New features (backwards compatible)
# PATCH - Bug fixes

# Create release tag
git tag -a v1.2.3 -m "Release v1.2.3 - Added backup feature"

# Push tag to remote
git push origin v1.2.3

# List tags
git tag -l

# Delete local tag
git tag -d v1.2.3

# Delete remote tag
git push origin :refs/tags/v1.2.3
```

```php
<?php
// Version information for module

/**
 * Module version constants
 */
define('YOURMODULE_VERSION', '1.2.3');
define('YOURMODULE_MIN_WHMCS_VERSION', '7.0.0');
define('YOURMODULE_RELEASE_DATE', '2024-01-15');

/**
 * Get module version info
 */
function yourmodule_getVersionInfo(): array
{
    return [
        'version' => YOURMODULE_VERSION,
        'min_whmcs_version' => YOURMODULE_MIN_WHMCS_VERSION,
        'release_date' => YOURMODULE_RELEASE_DATE,
        'changelog_url' => 'https://docs.example.com/changelog',
    ];
}

/**
 * Check WHMCS version compatibility
 */
function yourmodule_checkCompatibility(): bool
{
    $requiredVersion = YOURMODULE_MIN_WHMCS_VERSION;
    $currentVersion = \DI::make('config')->get('Version');

    return version_compare($currentVersion, $requiredVersion, '>=');
}
```

### Step 3: Module Packaging

```bash
#!/bin/bash
# scripts/package-module.sh

#!/usr/bin/env bash
set -e

MODULE_NAME="yourprovider"
VERSION=${1:-$(cat module.json | jq -r '.version')}
BUILD_DIR="release-build"
OUTPUT_DIR="releases"

echo "Packaging $MODULE_NAME v$VERSION"

# Clean up
rm -rf $BUILD_DIR
rm -f $OUTPUT_DIR/$MODULE_NAME-v$VERSION.zip

# Create build directory
mkdir -p $BUILD_DIR/modules/${MODULE_NAME}
mkdir -p $BUILD_DIR/assets
mkdir -p $OUTPUT_DIR

# Copy module files
rsync -av --exclude='.git' --exclude='tests/' --exclude='.idea/' \
    modules/servers/$MODULE_NAME/ $BUILD_DIR/modules/${MODULE_NAME}/

# Copy config files
cp module.json $BUILD_DIR/
cp README.md $BUILD_DIR/
cp CHANGELOG.md $BUILD_DIR/
cp LICENSE.md $BUILD_DIR/

# Copy assets if any
if [ -d "modules/servers/$MODULE_NAME/assets" ]; then
    cp -r modules/servers/$MODULE_NAME/assets $BUILD_DIR/assets/
fi

# Create version file
echo "{\"version\": \"$VERSION\", \"date\": \"$(date -u +%Y-%m-%d)\"}" > $BUILD_DIR/version.json

# Create package
cd $BUILD_DIR
zip -r ../$OUTPUT_DIR/$MODULE_NAME-v$VERSION.zip .
cd ..

# Clean up
rm -rf $BUILD_DIR

echo "Package created: $OUTPUT_DIR/$MODULE_NAME-v$VERSION.zip"
ls -la $OUTPUT_DIR/$MODULE_NAME-v$VERSION.zip
```

```json
// module.json - Package metadata
{
    "name": "yourprovider",
    "displayName": "YourProvider Hosting",
    "version": "1.2.3",
    "description": "Server provisioning module for YourProvider API",
    "author": {
        "name": "Your Company",
        "email": "support@example.com",
        "website": "https://example.com"
    },
    "license": "proprietary",
    "whmcsVersion": {
        "min": "7.0.0",
        "max": "8.x"
    },
    "dependencies": {},
    "files": [
        "modules/servers/yourprovider/",
        "assets/"
    ],
    "hooks": [
        "includes/hooks/custom_hooks.php"
    ],
    "configuration": [
        {
            "name": "api_key",
            "type": "text",
            "label": "API Key",
            "required": true
        }
    ],
    "install_instructions": [
        "1. Upload module to modules/servers/",
        "2. Navigate to Configuration > Servers",
        "3. Add New Server with YourProvider module",
        "4. Configure API credentials"
    ],
    "changelog": [
        "1.2.3 - Fixed backup scheduling issue",
        "1.2.2 - Added restore functionality",
        "1.2.1 - Improved error handling",
        "1.2.0 - Added automated backups"
    ]
}
```

### Step 4: Deployment Procedures

```bash
#!/bin/bash
# scripts/deploy-module.sh

#!/usr/bin/env bash
set -e

MODULE_NAME="yourprovider"
VERSION=${1}
ENVIRONMENT=${2:-staging}
REMOTE_USER=${3}
REMOTE_HOST=${4}

if [ -z "$VERSION" ] || [ -z "$REMOTE_HOST" ]; then
    echo "Usage: $0 <version> <environment> <user> <host>"
    exit 1
fi

PACKAGE="releases/$MODULE_NAME-v$VERSION.zip"
WHMCS_PATH="/var/www/whmcs"

echo "Deploying $MODULE_NAME v$VERSION to $ENVIRONMENT"

# Verify package exists
if [ ! -f "$PACKAGE" ]; then
    echo "Package not found: $PACKAGE"
    exit 1
fi

# Create backup of current installation
BACKUP_DIR="/var/backups/whmcs-modules/$(date +%Y%m%d-%H%M%S)"
ssh $REMOTE_USER@$REMOTE_HOST "mkdir -p $BACKUP_DIR"
ssh $REMOTE_USER@$REMOTE_HOST "cp -r $WHMCS_PATH/modules/servers/$MODULE_NAME $BACKUP_DIR/"

echo "Backup created at $BACKUP_DIR"

# Upload and extract package
scp $PACKAGE $REMOTE_USER@$REMOTE_HOST:/tmp/

ssh $REMOTE_USER@$REMOTE_HOST << EOF
    cd /tmp
    unzip -o $MODULE_NAME-v$VERSION.zip -d /tmp/module-upgrade/
    rsync -av --delete /tmp/module-upgrade/modules/servers/$MODULE_NAME/ $WHMCS_PATH/modules/servers/$MODULE_NAME/
    
    # Update permissions
    chown -R www-data:www-data $WHMCS_PATH/modules/servers/$MODULE_NAME/
    find $WHMCS_PATH/modules/servers/$MODULE_NAME -type d -exec chmod 755 {} \;
    find $WHMCS_PATH/modules/servers/$MODULE_NAME -type f -exec chmod 644 {} \;
    
    # Clear cache
    rm -rf $WHMCS_PATH/cache/templates_c/*/
    
    # Restart PHP-FPM
    systemctl restart php-fpm 2>/dev/null || true
    
    echo "Deployment complete"
EOF

# Clean up
ssh $REMOTE_USER@$REMOTE_HOST "rm -rf /tmp/module-upgrade /tmp/$MODULE_NAME-v$VERSION.zip"

echo "Deployment successful"
```

```bash
#!/bin/bash
# scripts/deploy-staging.sh

#!/usr/bin/env bash
set -e

MODULE_NAME="yourprovider"
VERSION=${1}

echo "Deploying to STAGING environment"

# Staging deployment targets
STAGING_HOSTS=(
    "staging1@example.com"
    "staging2@example.com"
)

for HOST in "${STAGING_HOSTS[@]}"; do
    echo "Deploying to $HOST..."
    ./deploy-module.sh $VERSION staging deploy $HOST
done

echo "Staging deployment complete"
echo "Run verification tests: ./verify-deployment.sh staging"
```

```bash
#!/bin/bash
# scripts/deploy-production.sh

#!/usr/bin/env bash
set -e

MODULE_NAME="yourprovider"
VERSION=${1}

# Production confirmation required
echo "WARNING: This will deploy to PRODUCTION"
echo "Module: $MODULE_NAME v$VERSION"
echo ""
read -p "Type 'deploy' to confirm: " confirm

if [ "$confirm" != "deploy" ]; then
    echo "Deployment cancelled"
    exit 1
fi

# Production deployment targets
PRODUCTION_HOSTS=(
    "prod1@example.com"
    "prod2@example.com"
    "prod3@example.com"
)

for HOST in "${PRODUCTION_HOSTS[@]}"; do
    echo "Deploying to $HOST..."
    ./deploy-module.sh $VERSION production admin $HOST
done

# Run post-deployment verification
./verify-deployment.sh production

echo "Production deployment complete"
```

### Step 5: Rollback Procedures

```bash
#!/bin/bash
# scripts/rollback-module.sh

#!/usr/bin/env bash
set -e

MODULE_NAME="yourprovider"
TARGET_VERSION=${1}
REMOTE_USER=${2}
REMOTE_HOST=${3}
WHMCS_PATH="/var/www/whmcs"

echo "Rolling back $MODULE_NAME to v$TARGET_VERSION"

# Find backup
BACKUP_DIR=$(ssh $REMOTE_USER@$REMOTE_HOST "ls -t /var/backups/whmcs-modules/ | head -1")
FULL_BACKUP_PATH="/var/backups/whmcs-modules/$BACKUP_DIR/$MODULE_NAME"

if [ ! -d "$FULL_BACKUP_PATH" ]; then
    echo "Backup not found. Attempting to find specific backup..."
    BACKUP_DIR=$(ssh $REMOTE_USER@$REMOTE_HOST "ls -t /var/backups/whmcs-modules/ | grep '$TARGET_VERSION' | head -1")
    FULL_BACKUP_PATH="/var/backups/whmcs-modules/$BACKUP_DIR"
fi

if [ -d "$FULL_BACKUP_PATH" ]; then
    echo "Found backup: $FULL_BACKUP_PATH"
else
    echo "ERROR: No suitable backup found"
    exit 1
fi

# Backup current version before rollback
CURRENT_BACKUP="/var/backups/whmcs-modules/pre-rollback-$(date +%Y%m%d-%H%M%S)"
ssh $REMOTE_USER@$REMOTE_HOST "cp -r $WHMCS_PATH/modules/servers/$MODULE_NAME $CURRENT_BACKUP/"

# Perform rollback
ssh $REMOTE_USER@$REMOTE_HOST << EOF
    rm -rf $WHMCS_PATH/modules/servers/$MODULE_NAME
    cp -r $FULL_BACKUP_PATH $WHMCS_PATH/modules/servers/$MODULE_NAME
    
    # Update permissions
    chown -R www-data:www-data $WHMCS_PATH/modules/servers/$MODULE_NAME/
    find $WHMCS_PATH/modules/servers/$MODULE_NAME -type d -exec chmod 755 {} \;
    find $WHMCS_PATH/modules/servers/$MODULE_NAME -type f -exec chmod 644 {} \;
    
    # Clear cache
    rm -rf $WHMCS_PATH/cache/templates_c/*/
    
    echo "Rollback complete"
EOF

echo "Rollback to v$TARGET_VERSION successful"
```

### Step 6: Post-Deployment Verification

```bash
#!/bin/bash
# scripts/verify-deployment.sh

#!/usr/bin/env bash
set -e

ENVIRONMENT=${1:-staging}
MODULE_NAME="your-provider"

echo "Running post-deployment verification for $ENVIRONMENT"

# Check 1: Module file exists
echo -n "Checking module files... "
if [ -d "modules/servers/$MODULE_NAME" ]; then
    echo "OK"
else
    echo "FAILED - Module files not found"
    exit 1
fi

# Check 2: Version check
echo -n "Checking module version... "
EXPECTED_VERSION=$(cat module.json | jq -r '.version')
# Would query WHMCS API here for actual installed version
echo "OK (v$EXPECTED_VERSION)"

# Check 3: File permissions
echo -n "Checking file permissions... "
PERMS_OK=$(find modules/servers/$MODULE_NAME -type f -perm -644 | wc -l)
TOTAL=$(find modules/servers/$MODULE_NAME -type f | wc -l)
if [ "$PERMS_OK" -eq "$TOTAL" ]; then
    echo "OK"
else
    echo "WARNING - Some files may have incorrect permissions"
fi

# Check 4: Module activation test
echo -n "Testing module activation... "
# Would test via WHMCS API here
echo "OK"

# Check 5: Run smoke tests
echo -n "Running smoke tests... "
./tests/smoke-tests.sh $ENVIRONMENT
echo "OK"

# Check 6: Error log check
echo -n "Checking for errors in log... "
if grep -q "Error" $WHMCS_LOG_DIR/module.log 2>/dev/null; then
    echo "WARNING - Errors found in log"
else
    echo "OK - No errors in log"
fi

echo ""
echo "Verification complete"
```

```php
<?php
// Module activation verification

function verifyModuleInstallation(): array
{
    $checks = [];

    // Check 1: Module files exist
    $modulePath = __DIR__ . '/modules/servers/yourprovider/';
    $checks['files_exist'] = [
        'name' => 'All module files present',
        'passed' => file_exists($modulePath . 'yourprovider.php'),
        'details' => 'Main module file exists',
    ];

    // Check 2: Version compatibility
    $checks['version_compatible'] = [
        'name' => 'WHMCS version compatible',
        'passed' => yourprovider_checkCompatibility(),
        'details' => 'Module compatible with installed WHMCS version',
    ];

    // Check 3: Configuration present
    $config = Capsule::table('tblserverconfig')
        ->where('type', 'yourprovider')
        ->first();

    $checks['config_exists'] = [
        'name' => 'Module configuration exists',
        'passed' => !empty($config),
        'details' => $config ? 'Server configured' : 'No server configuration found',
    ];

    // Check 4: Database tables created
    $checks['db_tables'] = [
        'name' => 'Database tables created',
        'passed' => Capsule::schema()->hasTable('mod_yourprovider_data'),
        'details' => 'Custom data table exists',
    ];

    // Check 5: Hooks registered
    $hooks = Capsule::table('tblhooks')
        ->where('hook', 'like', '%yourprovider%')
        ->count();

    $checks['hooks_registered'] = [
        'name' => 'Hooks properly registered',
        'passed' => $hooks > 0,
        'details' => "Found {$hooks} registered hooks",
    ];

    return $checks;
}

/**
 * Generate verification report
 */
function generateVerificationReport(array $checks): string
{
    $passed = 0;
    $failed = 0;

    $html = '<div class="verification-report">';
    $html .= '<h2>Deployment Verification Report</h2>';
    $html .= '<table class="table">';
    $html .= '<thead><tr><th>Check</th><th>Status</th><th>Details</th></tr></thead>';
    $html .= '<tbody>';

    foreach ($checks as $check) {
        $status = $check['passed'] ? 'passed' : 'failed';
        $icon = $check['passed'] ? 'check-circle' : 'times-circle';

        if ($check['passed']) {
            $passed++;
        } else {
            $failed++;
        }

        $html .= "<tr class=\"{$status}\">";
        $html .= "<td>{$check['name']}</td>";
        $html .= "<td><i class=\"fa fa-{$icon}\"></i> " . ($check['passed'] ? 'PASSED' : 'FAILED') . "</td>";
        $html .= "<td>{$check['details']}</td>";
        $html .= "</tr>";
    }

    $html .= '</tbody></table>';
    $html .= '<div class="summary">';
    $html .= "<p>Total: " . count($checks) . " | Passed: {$passed} | Failed: {$failed}</p>";
    $html .= '</div></div>';

    return $html;
}
```

### Step 7: CI/CD Pipeline Integration

```yaml
# .github/workflows/deploy.yml

name: Module Release Pipeline

on:
  push:
    tags:
      - 'v*'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Install Dependencies
        run: composer install

      - name: Run Tests
        run: vendor/bin/phpunit --testsuite Unit

      - name: Run Code Quality Check
        run: |
          vendor/bin/php-cs-fixer fix --dry-run --diff
          vendor/bin/phpmd modules text codesize,controversial,design,naming

  package:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Get Version
        id: version
        run: echo "::set-output name=version::${GITHUB_REF#refs/tags/v}"

      - name: Package Module
        run: ./scripts/package-module.sh ${{ steps.version.outputs.version }}

      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ github.event.repository.name }}-${{ steps.version.outputs.version }}
          path: releases/*.zip

  deploy-staging:
    needs: package
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3

      - name: Download Package
        uses: actions/download-artifact@v3
        with:
          name: ${{ github.event.repository.name }}-${{ github.ref_name }}
          path: /tmp

      - name: Deploy to Staging
        run: ./scripts/deploy-staging.sh ${{ github.ref_name }}

      - name: Verify Deployment
        run: ./scripts/verify-deployment.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url__: https://whmcs.example.com
    steps:
      - uses: actions/checkout@v3

      - name: Download Package
        uses: actions/download-artifact@v3
        with:
          name: ${{ github.event.repository.name }}-${{ github.ref_name }}
          path: /tmp

      - name: Deploy to Production
        run: ./scripts/deploy-production.sh ${{ github.ref_name }}

      - name: Verify Deployment
        run: ./scripts/verify-deployment.sh production

      - name: Notify Success
        if: success()
        run: ./scripts/notify-deployment.sh success production
```

## Deployment Best Practices

```
Pre-Deployment:
□ All tests passing in CI
□ Code review approved
□ Security scan passed
□ Staging deployment verified
□ Rollback procedure tested

Deployment:
□ Deploy during low-traffic periods
□ Use blue-green or rolling deployment
□ Monitor error rates closely
□ Keep deployment window small
□ Verify each step

Post-Deployment:
□ Run smoke tests
□ Monitor application logs
□ Check error tracking
□ Verify core functionality
□ Notify stakeholders

Rollback:
□ Automated rollback if health checks fail
□ Keep previous version accessible
□ Test rollback procedure regularly
□ Document rollback steps
```

## Verification Checklist

```
Deployment Package:
□ Version tag created
□ All source files included
□ Dependencies listed
□ Installation instructions complete
□ Changelog included

Environment:
□ Development verified
□ Staging deployment successful
□ Performance acceptable
□ No critical errors in logs

Production:
□ Deployment completed successfully
□ Module activates without errors
□ Core features functional
□ No errors in application logs
□ Performance metrics acceptable
□ Stakeholders notified
```

## WHMCS ClassDocs References

- [Module Development Guide](https://developers.whmcs.com/provisioning-modules/)
- [Module Lifecycle](https://developers.whmcs.com/provisioning-modules/module-lifecycle/)
- [Installation Guidelines](https://developers.whmcs.com/provisioning-modules/installation/)
- [Update Modules](https://developers.whmcs.com/provisioning-modules/updating/)
