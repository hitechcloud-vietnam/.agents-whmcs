# WHMCS CI/CD Pipeline Workflow

## Overview
This workflow establishes a continuous integration and deployment pipeline for WHMCS modules and customizations.

## Pipeline Architecture

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  Code   │────▶│  Build  │────▶│  Test   │────▶│ Deploy  │────▶│ Verify  │
│  Push   │     │ Stage   │     │ Stage   │     │ Stage   │     │ Stage   │
└─────────┘     └─────────┘     └─────────┘     └─────────┘     └─────────┘
     │              │              │              │              │
  Feature       Linting        Unit Tests    Staging        Smoke Tests
    Branch       Format        Integration   Production     Health Check
```

## Step 1: Git Repository Setup

```bash
# Initialize git repository
cd /var/www/whmcs/modules/addons/your_module
git init
git add .

# Create .gitignore
cat > .gitignore << 'EOF'
/vendor/
/node_modules/
/.phpunit.cache/
/coverage/
/storage/logs/*.log
/storage/module_cache/*.cache
.env
.env.*
!.env.example
composer.lock
*.swp
*.swo
.DS_Store
Thumbs.db
EOF

git commit -m "Initial commit"
git remote add origin https://github.com/org/whmcs-module.git
git push -u origin main
```

## Step 2: GitHub Actions CI/CD

```yaml
# .github/workflows/ci-cd.yml
name: WHMCS Module CI/CD

on:
  push:
    branches: [main, develop, 'release/**']
  pull_request:
    branches: [main]

env:
  PHP_VERSION: '8.1'
  COMPOSER_CACHE_DIR: ~/.composer/cache

jobs:
  # Job 1: Lint and Static Analysis
  lint:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: pdo_mysql, json, mbstring
          tools: composer, phpcs, phpstan, psalm

      - name: Cache Composer
        uses: actions/cache@v3
        with:
          path: ${{ env.COMPOSER_CACHE_DIR }}
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.json') }}

      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress

      - name: PHP CodeSniffer
        run: |
          ./vendor/bin/phpcs --standard=PSR12 \
            --colors \
            --warning-severity=0 \
            modules/

      - name: PHPStan
        run: |
          ./vendor/bin/phpstan analyse \
            modules/ \
            --level=max \
            --no-progress \
            --error-format=github

      - name: Psalm
        run: |
          ./vendor/bin/psalm --show-info=false

  # Job 2: Unit Tests
  test-unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: pdo_mysql, json, mbstring

      - name: Cache Composer
        uses: actions/cache@v3
        with:
          path: ${{ env.COMPOSER_CACHE_DIR }}
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.json') }}

      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run Unit Tests
        run: |
          ./vendor/bin/phpunit \
            --testsuite=Unit \
            --coverage-clover=coverage.xml \
            --colors=never

      - name: Upload Coverage
        if: github.event_name != 'pull_request'
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests

      - name: Upload Coverage PR
        if: github.event_name == 'pull_request'
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests
          token: ${{ secrets.CODECOV_TOKEN }}

  # Job 3: Integration Tests
  test-integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: whmcs_test
          MYSQL_USER: whmcs
          MYSQL_PASSWORD: whmcs
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: pdo_mysql, json, mbstring

      - name: Setup Test Database
        run: |
          mysql -h 127.0.0.1 -u root -proot -e "CREATE DATABASE IF NOT EXISTS whmcs_test"

      - name: Cache Composer
        uses: actions/cache@v3
        with:
          path: ${{ env.COMPOSER_CACHE_DIR }}
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.json') }}

      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run Integration Tests
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_NAME: whmcs_test
          DB_USER: root
          DB_PASSWORD: root
        run: |
          ./vendor/bin/phpunit \
            --testsuite=Integration \
            --colors=never

  # Job 4: Security Scan
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}

      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run Security Checker
        run: |
          composer require --dev roave/security-advisories
          ./vendor/bin/roave-security-advisories

      - name: Run PHP Security Checker
        run: |
          wget -q https://github.com/FriendsOfPHP/security-checker/releases/download/v7.1.3/security-checker.phar
          php security-checker.phar security:check composer.lock --format=github

  # Job 5: Deploy to Staging
  deploy-staging:
    name: Deploy to Staging
    needs: [lint, test-unit, test-integration, security]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Staging Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /var/www/whmcs-staging/modules/addons/your_module
            git pull origin develop
            composer install --no-dev --optimize-autoloader
            php artisan cache:clear
            chmod -R 755 .
            chmod -R 755 storage/

  # Job 6: Deploy to Production
  deploy-production:
    name: Deploy to Production
    needs: [deploy-staging]
    if: startsWith(github.ref, 'refs/heads/release/')
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Download Artifact
        uses: actions/download-artifact@v3
        with:
          name: module-package

      - name: Deploy to Production Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /var/www/whmcs/modules/addons/your_module
            # Backup current version
            tar -czf backup/$(date +%Y%m%d_%H%M%S)_your_module.tar.gz .
            # Deploy new version
            tar -xzf your_module.tar.gz
            composer install --no-dev --optimize-autoloader
            php artisan cache:clear
            # Run any pending migrations
            php your_module_migrate.php
            chmod -R 755 .
            chmod -R 755 storage/
```

## Step 3: Create Release Artifact

```yaml
# Job to add to ci-cd.yml for creating release packages
  package:
    name: Create Release Package
    needs: [lint, test-unit, test-integration, security]
    if: startsWith(github.ref, 'refs/heads/release/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}

      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress --no-dev

      - name: Create Package
        run: |
          tar -czvf your_module.tar.gz \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='tests' \
            --exclude='coverage' \
            --exclude='*.md' \
            --exclude='phpunit.xml*' \
            --exclude='.env*' \
            --exclude='.gitignore' \
            --exclude='.phpunit.cache' \
            .

      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: module-package
          path: your_module.tar.gz
```

## Step 4: Deployment Scripts

```bash
#!/bin/bash
# deploy.sh - Deploy WHMCS module to server

set -e

MODULE_NAME="your_module"
DEPLOY_PATH="/var/www/whmcs/modules/addons/${MODULE_NAME}"
BACKUP_PATH="/var/www/whmcs/backups/modules/${MODULE_NAME}"

echo "=== WHMCS Module Deployment ==="
echo "Module: ${MODULE_NAME}"
echo "Target: ${DEPLOY_PATH}"

# Create backup
echo "Creating backup..."
mkdir -p "${BACKUP_PATH}"
tar -czf "${BACKUP_PATH}/$(date +%Y%m%d_%H%M%S).tar.gz" -C "${DEPLOY_PATH}" .

# Deploy new version
echo "Deploying new version..."
rsync -avz --exclude='.git' --exclude='.env' ./ "${DEPLOY_PATH}/"

# Install dependencies
echo "Installing dependencies..."
cd "${DEPLOY_PATH}"
composer install --no-dev --optimize-autoloader

# Set permissions
echo "Setting permissions..."
chmod -R 755 "${DEPLOY_PATH}"
chmod -R 755 "${DEPLOY_PATH}/storage/"

# Clear cache
echo "Clearing cache..."
cd /var/www/whmcs
php artisan cache:clear

# Run module migrations
echo "Running migrations..."
php "${DEPLOY_PATH}/${MODULE_NAME}_migrate.php"

echo "=== Deployment Complete ==="
```

## Step 5: Rollback Script

```bash
#!/bin/bash
# rollback.sh - Rollback WHMCS module to previous version

set -e

MODULE_NAME="your_module"
DEPLOY_PATH="/var/www/whmcs/modules/addons/${MODULE_NAME}"
BACKUP_PATH="/var/www/whmcs/backups/modules/${MODULE_NAME}"

echo "=== WHMCS Module Rollback ==="

# List available backups
echo "Available backups:"
ls -la "${BACKUP_PATH}/"

# Get latest backup
LATEST_BACKUP=$(ls -t "${BACKUP_PATH}" | head -1)

if [ -z "${LATEST_BACKUP}" ]; then
    echo "No backup found!"
    exit 1
fi

echo "Rolling back to: ${LATEST_BACKUP}"

# Restore backup
rm -rf "${DEPLOY_PATH}"
mkdir -p "${DEPLOY_PATH}"
tar -xzf "${BACKUP_PATH}/${LATEST_BACKUP}" -C "${DEPLOY_PATH}"

# Clear cache
echo "Clearing cache..."
cd /var/www/whmcs
php artisan cache:clear

echo "=== Rollback Complete ==="
```

## Step 6: Environment Configuration

```bash
# .env.staging
APP_ENV=staging
APP_DEBUG=true
DB_HOST=staging-db.internal
DB_NAME=whmcs_staging
DB_USER=whmcs_staging
DB_PASSWORD=secure_password_here
MODULE_API_KEY=test_api_key
MODULE_WEBHOOK_URL=https://staging-webhook.example.com

# .env.production
APP_ENV=production
APP_DEBUG=false
DB_HOST=prod-db.internal
DB_NAME=whmcs_production
DB_USER=whmcs_production
DB_PASSWORD=secure_password_here
MODULE_API_KEY=production_api_key
MODULE_WEBHOOK_URL=https://production-webhook.example.com
```

## Step 7: Pre-Deployment Checks

```php
<?php
// pre-deploy-check.php

echo "=== Pre-Deployment Checks ===\n\n";

$checks = [
    'Database Connection' => function() {
        try {
            Capsule::connection()->getPdo();
            return true;
        } catch (\Exception $e) {
            return "Failed: " . $e->getMessage();
        }
    },
    'Module Directory Writable' => function() {
        $dir = dirname(__DIR__) . '/modules/addons/your_module';
        return is_writable($dir) ? true : "Not writable";
    },
    'Storage Directory Writable' => function() {
        $dir = dirname(__DIR__) . '/storage/logs';
        return is_writable($dir) ? true : "Not writable";
    },
    'Config Table Exists' => function() {
        $tables = Capsule::connection()->getDoctrineSchemaManager()->listTableNames();
        return in_array('tblconfiguration', $tables) ? true : "Missing";
    },
    'PHP Version' => function() {
        return PHP_VERSION_ID >= 80100 ? true : "PHP 8.1+ required";
    },
    'Required Extensions' => function() {
        $required = ['pdo_mysql', 'json', 'mbstring', 'curl'];
        $missing = [];
        foreach ($required as $ext) {
            if (!extension_loaded($ext)) {
                $missing[] = $ext;
            }
        }
        return empty($missing) ? true : "Missing: " . implode(', ', $missing);
    },
    'Disk Space' => function() {
        $free = disk_free_space('/');
        $freeGB = round($free / 1024 / 1024 / 1024, 2);
        return $freeGB > 1 ? true : "Low: {$freeGB}GB";
    }
];

$allPassed = true;
foreach ($checks as $name => $check) {
    $result = $check();
    $status = $result === true ? "PASS" : "FAIL";
    if ($result !== true) {
        $allPassed = false;
    }
    echo "[{$status}] {$name}";
    if ($result !== true) {
        echo " - {$result}";
    }
    echo "\n";
}

echo "\n";
if ($allPassed) {
    echo "All checks passed. Ready to deploy.\n";
    exit(0);
} else {
    echo "Some checks failed. Please resolve issues before deploying.\n";
    exit(1);
}
```

## Step 8: Post-Deployment Verification

```yaml
# Add to deployment job
- name: Post-Deployment Verification
  run: |
    # Wait for deployment to settle
    sleep 10

    # Check module is accessible
    curl -f https://staging.example.com/whmcs/admin/addonmodules.php?module=your_module
    echo "Module admin page accessible"

    # Run smoke tests
    ./vendor/bin/phpunit tests/Smoke

    # Check for errors in logs
    ssh ${{ secrets.STAGING_HOST }} "tail -50 /var/www/whmcs/storage/logs/module_debug.log | grep -i error"

    # Verify database schema
    ssh ${{ secrets.STAGING_HOST }} "mysql -u root -e 'DESCRIBE whmcs_staging.mod_your_module'"

- name: Notify Deployment
  if: always()
  uses: slackapi/slack-github-action@v1
  with:
    channel-id: 'deployment-alerts'
    payload: |
      {
        "text": "Deployment ${{ job.status }}: ${{ github.ref }} by ${{ github.actor }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*WHMCS Module Deployment*\n*Status:* ${{ job.status }}\n*Branch:* ${{ github.ref }}\n*Actor:* ${{ github.actor }}"
            }
          }
        ]
      }
```

## Verification Checklist

- [ ] Git repository initialized
- [ ] GitHub Actions workflow created
- [ ] Linting job configured
- [ ] Unit test job configured
- [ ] Integration test job with database
- [ ] Security scan job configured
- [ ] Staging deployment job configured
- [ ] Production deployment job configured
- [ ] Release artifact creation configured
- [ ] Deployment script created
- [ ] Rollback script created
- [ ] Pre-deployment checks implemented
- [ ] Post-deployment verification configured
- [ ] Slack notifications configured
- [ ] Environment secrets configured
