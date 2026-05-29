# WHMCS Staging Deployment Workflow

## Overview
This workflow covers deploying WHMCS modules and customizations to a staging environment for testing before production release.

## Pre-Deployment Phase

### Step 1: Pre-Deployment Checklist

```bash
#!/bin/bash
# pre-staging-check.sh

echo "=== Pre-Staging Deployment Checks ==="

# Check 1: All tests passing
echo "Running tests..."
./vendor/bin/phpunit --testsuite=Unit,Integration
if [ $? -ne 0 ]; then
    echo "FAIL: Tests failed"
    exit 1
fi
echo "PASS: All tests passing"

# Check 2: Code style compliance
echo "Checking code style..."
./vendor/bin/phpcs --standard=PSR12 modules/
if [ $? -ne 0 ]; then
    echo "FAIL: Code style issues found"
    exit 1
fi
echo "PASS: Code style compliant"

# Check 3: Security scan
echo "Running security scan..."
./vendor/bin/roave-security-advisories
if [ $? -ne 0 ]; then
    echo "FAIL: Security vulnerabilities found"
    exit 1
fi
echo "PASS: No known security vulnerabilities"

# Check 4: Static analysis
echo "Running static analysis..."
./vendor/bin/phpstan analyse modules/ --level=max
if [ $? -ne 0 ]; then
    echo "FAIL: Static analysis found issues"
    exit 1
fi
echo "PASS: Static analysis clean"

echo "=== All Pre-Deployment Checks Passed ==="
```

### Step 2: Create Staging Build

```bash
#!/bin/bash
# create-staging-build.sh

VERSION=${1:-"staging-$(date +%Y%m%d-%H%M%S)"}
BUILD_DIR="./builds/${VERSION}"
MODULE_NAME="your_module"

echo "Creating staging build: ${VERSION}"

# Create build directory
mkdir -p ${BUILD_DIR}

# Copy module files
rsync -av \
    --exclude='.git' \
    --exclude='.github' \
    --exclude='tests' \
    --exclude='coverage' \
    --exclude='*.md' \
    --exclude='phpunit.xml*' \
    --exclude='.env*' \
    --exclude='.gitignore' \
    --exclude='.phpunit.cache' \
    --exclude='node_modules' \
    ./modules/addons/${MODULE_NAME}/ ${BUILD_DIR}/${MODULE_NAME}/

# Copy composer files
cp composer.json ${BUILD_DIR}/
cp composer.lock ${BUILD_DIR}/

# Create staging configuration
cat > ${BUILD_DIR}/.env.staging << 'EOF'
APP_ENV=staging
MODULE_DEBUG=true
MODULE_LOG_LEVEL=debug
DB_HOST=staging-db.internal
DB_NAME=whmcs_staging
API_ENDPOINT=https://staging-api.example.com
WEBHOOK_URL=https://staging-webhook.example.com
EOF

# Install production dependencies
cd ${BUILD_DIR}
composer install --no-dev --optimize-autoloader

# Create archive
tar -czvf ${BUILD_DIR}.tar.gz -C builds ${VERSION}

# Clean up build directory
rm -rf ${BUILD_DIR}

echo "Build created: builds/${VERSION}.tar.gz"
```

## Staging Server Setup

### Step 3: Configure Staging Server

```bash
#!/bin/bash
# setup-staging-server.sh

STAGING_HOST="staging.whmcs.example.com"
STAGING_USER="deploy"
MODULE_NAME="your_module"
WHMCS_PATH="/var/www/whmcs-staging"

echo "=== Setting up Staging Server ==="

# SSH into staging server
ssh ${STAGING_USER}@${STAGING_HOST} << 'EOF'
    # Create staging WHMCS directory
    sudo mkdir -p ${WHMCS_PATH}
    sudo chown -R www-data:www-data ${WHMCS_PATH}

    # Create module directory
    mkdir -p ${WHMCS_PATH}/modules/addons/${MODULE_NAME}

    # Create storage directories
    mkdir -p ${WHMCS_PATH}/storage/{logs,module_cache,temp}
    chmod -R 777 ${WHMCS_PATH}/storage

    # Create backup directory
    mkdir -p ${WHMCS_PATH}/backups/modules/${MODULE_NAME}

    # Copy staging license
    # (You'll need to get a staging license from WHMCS)
EOF

echo "Staging server configured"
```

### Step 4: Database Setup for Staging

```sql
-- staging-database-setup.sql

-- Create staging database
CREATE DATABASE whmcs_staging CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user
CREATE USER 'whmcs_staging'@'%' IDENTIFIED BY 'staging_password_here';
GRANT ALL PRIVILEGES ON whmcs_staging.* TO 'whmcs_staging'@'%';
FLUSH PRIVILEGES;

-- Create module-specific tables
CREATE TABLE IF NOT EXISTS `mod_your_module` (
    `id` INT NOT NULL AUTO_INCREMENT,
    `user_id` INT DEFAULT NULL,
    `config` TEXT,
    `status` VARCHAR(50) DEFAULT 'pending',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `idx_user_id` (`user_id`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create staging-specific configuration
INSERT INTO `tblconfiguration` (`setting`, `value`, `friendlyname`, `description`, `order`, `type`) VALUES
('ModuleStagingMode', 'enabled', 'Module Staging Mode', 'Enable staging mode for testing', 100, 'yesno');
```

## Deployment Phase

### Step 5: Deploy to Staging

```bash
#!/bin/bash
# deploy-to-staging.sh

VERSION=${1:-"staging"}
STAGING_HOST="staging.whmcs.example.com"
STAGING_USER="deploy"
MODULE_NAME="your_module"
BUILD_FILE="builds/staging.tar.gz"

echo "=== Deploying to Staging ==="

# Create backup of current staging version
echo "Creating backup..."
ssh ${STAGING_USER}@${STAGING_HOST} "
    cd /var/www/whmcs-staging/modules/addons/${MODULE_NAME}
    if [ -d .git ]; then
        BACKUP_NAME=/var/www/whmcs-staging/backups/modules/${MODULE_NAME}/staging-$(date +%Y%m%d-%H%M%S).tar.gz
        tar -czf $BACKUP_NAME .
        echo 'Backup created: $BACKUP_NAME'
    fi
"

# Upload new build
echo "Uploading build..."
scp ${BUILD_FILE} ${STAGING_USER}@${STAGING_HOST}:/tmp/staging.tar.gz

# Extract and deploy
echo "Deploying..."
ssh ${STAGING_USER}@${STAGING_HOST} << 'EOF'
    cd /var/www/whmcs-staging/modules/addons/${MODULE_NAME}

    # Remove old files
    rm -rf *

    # Extract new build
    tar -xzf /tmp/staging.tar.gz

    # Set permissions
    chown -R www-data:www-data .
    find . -type d -exec chmod 755 {} \;
    find . -type f -exec chmod 644 {} \;
    chmod -R 777 storage/ 2>/dev/null || true

    # Clear WHMCS cache
    cd /var/www/whmcs-staging
    php artisan cache:clear 2>/dev/null || true

    # Run module migrations
    php modules/addons/your_module/migrations/run.php staging

    # Clean up
    rm /tmp/staging.tar.gz

    echo "Deployment complete"
EOF

echo "=== Deployment to Staging Complete ==="
```

### Step 6: Post-Deployment Verification

```bash
#!/bin/bash
# verify-staging-deployment.sh

STAGING_URL="https://staging.whmcs.example.com"
STAGING_USER="deploy"
STAGING_HOST="staging.whmcs.example.com"

echo "=== Post-Deployment Verification ==="

# Check 1: Module files exist
echo "Checking module files..."
ssh ${STAGING_USER}@${STAGING_HOST} "test -f /var/www/whmcs-staging/modules/addons/your_module/your_module.php" && echo "PASS: Main file exists" || echo "FAIL: Main file missing"

# Check 2: Module activation
echo "Checking module activation status..."
curl -s "${STAGING_URL}/whmcs/admin/addonmodules.php?module=your_module" | grep -q "Your Module" && echo "PASS: Module accessible" || echo "FAIL: Module not accessible"

# Check 3: Database tables
echo "Checking database tables..."
ssh ${STAGING_USER}@${STAGING_HOST} "
    mysql -u whmcs_staging -pwhmcs_staging whmcs_staging -e 'SHOW TABLES LIKE \"mod_your_module\";'
" && echo "PASS: Database table exists" || echo "FAIL: Database table missing"

# Check 4: Error logs
echo "Checking error logs..."
ERROR_COUNT=$(ssh ${STAGING_USER}@${STAGING_HOST} "tail -100 /var/www/whmcs-staging/storage/logs/module_error.log | grep -i error | wc -l")
if [ "$ERROR_COUNT" -eq 0 ]; then
    echo "PASS: No errors in logs"
else
    echo "WARN: Found $ERROR_COUNT potential errors in logs"
fi

# Check 5: API endpoints
echo "Testing API endpoints..."
curl -s "${STAGING_URL}/whmcs/api.php?action=GetModuleAPIVersion&username=admin&password=$(cat ~/.whmcs_staging_pass)&responsetype=json" | jq -r '.status' 2>/dev/null | grep -q success && echo "PASS: API working" || echo "FAIL: API not responding"

echo "=== Verification Complete ==="
```

## Staging Testing

### Step 7: Automated Staging Tests

```yaml
# .github/workflows/staging-tests.yml
name: Staging Tests

on:
  workflow_run:
    workflows: ["Deploy to Staging"]
    types: [completed]

jobs:
  staging-smoke-tests:
    name: Smoke Tests on Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: pdo_mysql, json

      - name: Install dependencies
        run: composer install --no-dev

      - name: Run Smoke Tests
        run: |
          php vendor/bin/phpunit tests/Smoke \
            --config=phpunit-staging.xml
        env:
          WHMCS_STAGING_URL: ${{ secrets.STAGING_URL }}
          WHMCS_STAGING_API_KEY: ${{ secrets.STAGING_API_KEY }}

      - name: Module Functionality Tests
        run: |
          php tests/staging/module-test.php
        env:
          STAGING_URL: ${{ secrets.STAGING_URL }}
          STAGING_USERNAME: ${{ secrets.STAGING_ADMIN_USER }}
          STAGING_PASSWORD: ${{ secrets.STAGING_ADMIN_PASSWORD }}

      - name: Integration Tests
        run: |
          php tests/staging/integration-test.php
        env:
          STAGING_URL: ${{ secrets.STAGING_URL }}
          STAGING_API_USER: ${{ secrets.STAGING_API_USER }}
          STAGING_API_KEY: ${{ secrets.STAGING_API_KEY }}

      - name: Check Error Logs
        run: |
          ssh ${{ secrets.STAGING_HOST }} "tail -50 /var/www/whmcs-staging/storage/logs/module_error.log" || echo "No errors found"
```

### Step 8: Manual Testing Checklist

```markdown
## Staging Manual Testing Checklist

### Module Installation
- [ ] Module appears in WHMCS Admin > Extensions
- [ ] Module can be activated without errors
- [ ] Module configuration page loads
- [ ] Settings can be saved

### Client Functionality
- [ ] Client can access module features
- [ ] Client data is correctly displayed
- [ ] Actions can be performed by clients
- [ ] Email notifications sent correctly

### Admin Functionality
- [ ] Admin dashboard shows module status
- [ ] Admin can view client data
- [ ] Admin can modify settings
- [ ] Admin can perform bulk operations

### API Integration
- [ ] API authentication works
- [ ] API endpoints return correct data
- [ ] API error handling works
- [ ] Rate limiting works correctly

### Webhooks
- [ ] Webhooks fire on events
- [ ] Webhook payloads are correct
- [ ] Webhook retry logic works

### Performance
- [ ] Page load times acceptable
- [ ] No N+1 queries
- [ ] Caching works correctly
- [ ] Memory usage acceptable

### Security
- [ ] CSRF protection works
- [ ] Permission checks work
- [ ] Input validation works
- [ ] SQL injection prevented

### Compatibility
- [ ] Works with WHMCS v8.0
- [ ] Works with WHMCS v8.1
- [ ] Works with WHMCS v8.2
- [ ] Works with PHP 8.0
- [ ] Works with PHP 8.1
```

## Approval Workflow

### Step 9: Request Staging Approval

```markdown
## Staging Deployment Approval Request

### Deployment Details
- **Version**: v1.2.0
- **Deployed by**: @developer
- **Deployed at**: 2024-01-15 14:30 UTC
- **Staging URL**: https://staging.whmcs.example.com

### Changes Included
- Feature: New reporting dashboard
- Feature: Enhanced API rate limiting
- Fix: Corrected timezone handling
- Fix: Resolved caching issue

### Test Results
- Unit Tests: 45/45 passed
- Integration Tests: 12/12 passed
- Smoke Tests: 8/8 passed
- Security Scan: No vulnerabilities found

### Manual Testing
- [ ] Module installation tested
- [ ] Client functionality tested
- [ ] Admin functionality tested
- [ ] API integration tested
- [ ] Performance tested

### Pre-Production Checklist
- [ ] All tests passing
- [ ] No critical errors in logs
- [ ] Documentation updated
- [ ] Rollback plan documented
- [ ] Production window scheduled

### Approval Required
Please review and approve for production deployment.

Reviewers: @senior-dev @qa-lead
```

### Step 10: Staging to Production Promotion

```bash
#!/bin/bash
# promote-to-production.sh

STAGING_HOST="staging.whmcs.example.com"
PRODUCTION_HOST="whmcs.example.com"
STAGING_USER="deploy"
PRODUCTION_USER="deploy"
MODULE_NAME="your_module"
VERSION=${1}

if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    exit 1
fi

echo "=== Promoting Version ${VERSION} to Production ==="

# Verify version exists
if [ ! -f "builds/${VERSION}.tar.gz" ]; then
    echo "Build not found: builds/${VERSION}.tar.gz"
    exit 1
fi

# Verify staging deployment matches version
echo "Verifying staging deployment..."
ssh ${STAGING_USER}@${STAGING_HOST} "cat /var/www/whmcs-staging/modules/addons/${MODULE_NAME}/VERSION" | grep -q ${VERSION}
if [ $? -ne 0 ]; then
    echo "WARNING: Staging version doesn't match requested version"
    read -p "Continue anyway? (y/n) " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 1
    fi
fi

# Create production backup
echo "Creating production backup..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    cd /var/www/whmcs/modules/addons/${MODULE_NAME}
    BACKUP_NAME=/var/www/whmcs/backups/modules/${MODULE_NAME}/production-$(date +%Y%m%d-%H%M%S)-v${VERSION}.tar.gz
    tar -czf $BACKUP_NAME .
    echo 'Production backup created: $BACKUP_NAME'
"

# Upload to production
echo "Uploading to production..."
scp builds/${VERSION}.tar.gz ${PRODUCTION_USER}@${PRODUCTION_HOST}:/tmp/production.tar.gz

# Deploy to production
echo "Deploying to production..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << 'EOF'
    cd /var/www/whmcs/modules/addons/your_module

    # Deploy new version
    tar -xzf /tmp/production.tar.gz

    # Set permissions
    chown -R www-data:www-data .
    find . -type d -exec chmod 755 {} \;
    find . -type f -exec chmod 644 {} \;
    chmod -R 777 storage/ 2>/dev/null || true

    # Clear cache
    cd /var/www/whmcs
    php artisan cache:clear

    # Run migrations
    php modules/addons/your_module/migrations/run.php production

    # Clean up
    rm /tmp/production.tar.gz

    echo "Production deployment complete"
EOF

echo "=== Version ${VERSION} Promoted to Production ==="
```

## Verification Checklist

- [ ] Pre-deployment checks completed
- [ ] Staging build created
- [ ] Staging server configured
- [ ] Database setup complete
- [ ] Module deployed to staging
- [ ] Post-deployment verification passed
- [ ] Automated smoke tests passed
- [ ] Manual testing completed
- [ ] Approval received
- [ ] Production backup created
- [ ] Version promoted to production
- [ ] Production verification completed
