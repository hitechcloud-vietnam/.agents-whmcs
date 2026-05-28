# WHMCS CI/CD Workflow

## Purpose

Automated pipeline for building, testing, and deploying WHMCS customizations and modules. Ensures consistent, reliable deployments with automated quality gates.

## Prerequisites

- Git repository for WHMCS code
- CI/CD platform (GitHub Actions, GitLab CI, Jenkins)
- Docker (optional, for containerized builds)
- Deployment targets (staging, production)
- Secrets management

## Workflow Steps

### Step 1: Project Structure Setup

Organize WHMCS customizations for version control:

```bash
# Recommended project structure
whmcs-custom-modules/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── .gitlab-ci.yml
├── composer.json
├── phpcs.xml
├── phpstan.neon
├── modules/
│   └── custom/
│       └── your_module/
│           ├── your_module.php
│           ├── hooks.php
│           ├── includes/
│           └── templates/
├── hooks/
│   └── custom_hooks.php
├── resources/
│   └── testing/
│       └── Unit/
└── scripts/
    └── deployment/
```

Configure composer.json:

```json
{
    "name": "company/whmcs-customizations",
    "description": "Custom WHMCS modules and hooks",
    "type": "project",
    "require": {
        "php": "^7.4|^8.0",
        "whmcs/whmcs": "^8.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^9.5",
        "squizlabs/php_codesniffer": "^3.6",
        "phpstan/phpstan": "^1.0"
    },
    "autoload": {
        "psr-4": {
            "Company\\WHMCSExtensions\\": "modules/custom/"
        }
    },
    "scripts": {
        "test": "php vendor/bin/phpunit",
        "cs-check": "php vendor/bin/phpcs --standard=phpcs.xml modules/",
        "cs-fix": "php vendor/bin/phpcbf --standard=phpcs.xml modules/",
        "analyse": "php vendor/bin/phpstan analyse modules/ --level=5"
    }
}
```

### Step 2: Configure Static Analysis

Set up code quality tools:

```xml
<!-- phpcs.xml -->
<?xml version="1.0"?>
<ruleset name="WHMCS Custom Code Standards">
    <description>WHMCS Custom Modules Coding Standards</description>
    
    <file>modules/</file>
    <file>hooks/</file>
    
    <exclude-pattern>*/vendor/*</exclude-pattern>
    
    <rule ref="PSR12">
        <exclude name="PSR1.Methods.CamelCapsMethodName"/>
    </rule>
    
    <rule ref="Generic.Arrays.DisallowShortArraySyntax"/>
    
    <properties>
        <property name="tabWidth" value="4"/>
        <property name="indentWithSpaces" value="true"/>
    </properties>
</ruleset>
```

```yaml
# phpstan.neon
parameters:
    level: 5
    paths:
        - modules/
        - hooks/
    excludePaths:
        - vendor/
    ignoreErrors:
        - '#Call to undefined method.*#'
        - '#Constant .* not found#'
    reportUnmatchedIgnoredErrors: false
```

### Step 3: Create Unit Tests

Set up testing framework:

```php
<?php
// resources/testing/bootstrap.php
require_once __DIR__ . '/../../vendor/autoload.php';
require_once __DIR__ . '/../../init.php';

define('WHMCS_LARGE_MODE', true);

// Use testing database if available
if (getenv('TESTING_MODE')) {
    // Override database connection
}
```

```php
<?php
// resources/testing/Unit/ModuleTest.php
namespace WHMCS\Testing\Unit;

use PHPUnit\Framework\TestCase;

class CustomModuleTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
    }
    
    public function testModuleConfigReturnsArray()
    {
        $module = new \Company\YourModule\YourModule();
        $config = $module->getConfigArray();
        
        $this->assertIsArray($config);
        $this->assertArrayHasKey('name', $config);
    }
    
    public function testHookRegistration()
    {
        $this->assertTrue(
            function_exists('add_hook'),
            'WHMCS add_hook function should be available'
        );
    }
    
    public function testClientValidationLogic()
    {
        $validator = new \Company\YourModule\Validator();
        
        $this->assertTrue($validator->isValidEmail('test@example.com'));
        $this->assertFalse($validator->isValidEmail('invalid-email'));
    }
}
```

```xml
<!-- phpunit.xml -->
<?xml version="1.0"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="resources/testing/bootstrap.php"
         colors="true"
         cacheDirectory=".phpunit.cache">
    <testsuites>
        <testsuite name="Unit">
            <directory>resources/testing/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>resources/testing/Integration</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">modules/</directory>
        </include>
    </coverage>
</phpunit>
```

### Step 4: Configure GitHub Actions

Create CI workflow:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: dom, curl, libxml, mbstring, zip, gd
          coverage: xdebug
        
      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress
      
      - name: PHP CodeSniffer
        run: composer cs-check || composer cs-fix && git diff --exit-code
      
      - name: PHPStan
        run: composer analyse || true
      
      - name: Register Comment
        if: github.event_name == 'pull_request'
        uses:机器actions/github-script@v6
        with:
          script: |
            // Post analysis results as PR comment

  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: whmcs_test
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
      
      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress
      
      - name: Run Unit Tests
        env:
          TESTING_MODE: true
          DB_HOST: 127.0.0.1
          DB_DATABASE: whmcs_test
          DB_USERNAME: root
          DB_PASSWORD: root
        run: composer test -- --coverage-text
      
      - name: Upload Coverage
        if: success()
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
```

### Step 5: Configure Deployment Pipeline

Create deployment workflow:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    name: Deploy to ${{ github.event.inputs.environment || 'staging' }}
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment || 'staging' }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
      
      - name: Install Dependencies
        run: composer install --no-dev --optimize-autoloader
      
      - name: Create Deployment Package
        run: |
          mkdir -p dist
          tar -czf dist/deploy.tar.gz \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='node_modules' \
            --exclude='*.md' \
            --exclude='phpunit.xml*' \
            --exclude='phpstan.neon*' \
            --exclude='composer.lock' \
            .
      
      - name: Deploy to Server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          passphrase: ${{ secrets.SERVER_SSH_PASSPHRASE }}
          script: |
            cd /var/www/whmcs/modules/custom
            # Backup current version
            if [ -d your_module ]; then
              tar -czf /var/backups/your_module-$(date +%Y%m%d).tar.gz your_module/
              rm -rf your_module
            fi
            
            # Extract new version
            tar -xzf /tmp/deploy.tar.gz
            
            # Set permissions
            chown -R www-data:www-data your_module/
            find your_module/ -type f -name "*.php" -exec chmod 644 {} \;
            find your_module/ -type d -exec chmod 755 {} \;
            
            # Clear cache
            php /var/www/whmcs/crons/cron.php?a=clearCache
      
      - name: Run Smoke Tests
        run: |
          curl -f https://staging.example.com/modules/custom/your_module/healthcheck.php
          curl -f https://staging.example.com/admin/healthcheck.php
      
      - name: Notify Success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'deployments'
          payload: |
            {
              "text": "Deployment successful!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Successful*\nEnvironment: staging\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
```

### Step 6: Database Migration Pipeline

Handle database changes safely:

```yaml
# .github/workflows/database.yml
name: Database Migrations

jobs:
  migration-check:
    name: Validate Migrations
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
      
      - name: Run Migration Dry Run
        run: |
          php resources/database/migrate.php --dry-run --target=latest
      
      - name: Generate Migration Plan
        if: always()
        run: |
          php resources/database/migrate.php --plan > migration_plan.txt
          cat migration_plan.txt
```

```php
<?php
// resources/database/migrate.php
// Database migration script

class DatabaseMigration
{
    private $migrations = [
        '20240101_add_custom_field' => 'Up_20240101_add_custom_field',
        '20240115_add_custom_index' => 'Up_20240115_add_custom_index',
    ];
    
    public function run($direction = 'up', $target = null)
    {
        $currentVersion = $this->getCurrentVersion();
        
        if ($direction === 'up') {
            $this->migrateUp($target);
        } else {
            $this->migrateDown($target);
        }
    }
    
    public function dryRun()
    {
        // Show what would be executed without running
        $pending = $this->getPendingMigrations();
        foreach ($pending as $version => $migration) {
            echo "Would run: $version - {$migration['description']}\n";
        }
    }
    
    private function migrateUp($target = null)
    {
        $pending = $this->getPendingMigrations();
        
        foreach ($pending as $version => $class) {
            if ($target && $version > $target) break;
            
            echo "Running migration: $version\n";
            $this->$class();
            $this->recordMigration($version);
        }
    }
    
    private function Up_20240101_add_custom_field()
    {
        Capsule::schema()->table('tblclients', function ($Blueprint) {
            $Blueprint->string('custom_field', 255)->nullable();
        });
    }
}
```

### Step 7: Set Up Environment Configuration

Manage environment-specific settings:

```bash
# .env.staging
APP_ENV=staging
APP_DEBUG=true
DB_HOST=staging-db.example.com
DB_DATABASE=whmcs_staging
SECURE_COOKIES=false

# .env.production
APP_ENV=production
APP_DEBUG=false
DB_HOST=prod-db.example.com
DB_DATABASE=whmcs_production
SECURE_COOKIES=true
```

```php
<?php
// includes/custom_config.php
// Load environment-specific configuration

$envFile = __DIR__ . '/../.env.' . (getenv('APP_ENV') ?: 'staging');

if (file_exists($envFile)) {
    $lines = file($envFile, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
    foreach ($lines as $line) {
        if (strpos($line, '=') !== false) {
            list($key, $value) = explode('=', $line, 2);
            $key = trim($key);
            $value = trim($value);
            if (!getenv($key)) {
                putenv("$key=$value");
            }
        }
    }
}
```

### Step 8: Configure Rollback Strategy

Implement safe rollback:

```yaml
# .github/workflows/rollback.yml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    name: Rollback to ${{ github.event.inputs.version }}
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Confirm Rollback
        run: |
          echo "Rolling back to version: ${{ github.event.inputs.version }}"
          echo "This will restore from backup: /var/backups/your_module-${{ github.event.inputs.version }}.tar.gz"
      
      - name: Deploy Previous Version
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /var/www/whmcs/modules/custom
            
            # Backup current broken version
            tar -czf /var/backups/your_module-broken-$(date +%Y%m%d).tar.gz your_module/
            
            # Restore previous version
            rm -rf your_module
            tar -xzf /var/backups/your_module-${{ github.event.inputs.version }}.tar.gz
            
            # Set permissions
            chown -R www-data:www-data your_module/
            
            # Clear cache
            php /var/www/whmcs/crons/cron.php?a=clearCache
            
            # Verify
            php /var/www/whmcs/modules/custom/your_module/healthcheck.php
```

## Verification Checklist

- [ ] Repository structure organized correctly
- [ ] All CI/CD tools configured and tested
- [ ] Unit tests created and passing
- [ ] Static analysis configured
- [ ] Deployment pipeline tested on staging
- [ ] Rollback procedure tested
- [ ] Secrets properly configured
- [ ] Notifications configured
- [ ] Documentation updated
- [ ] Team trained on deployment process

## Related Skills and Documentation

- [WHMCS Deployment Best Practices](whmcs-deployment-best-practices.md)
- [WHMCS Module Testing](whmcs-module-testing.md)
- [WHMCS Code Review](whmcs-code-review-workflow.md)
- WHMCS GitHub Actions: https://developers.whmcs.com/advanced/automating-with-github-actions/

## Notes

- Always test deployments on staging first
- Keep deployment packages for quick rollback
- Use semantic versioning for releases
- Monitor deployments after release
- Document any manual steps required
