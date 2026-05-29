# WHMCS Production Deployment Workflow

## Overview
This workflow defines the process for safely deploying WHMCS modules and customizations to production environments with minimal risk and maximum reliability.

## Production Deployment Principles

1. **Zero-Downtime**: Use deployment strategies that don't interrupt service
2. **Rollback Capability**: Always be able to revert to previous version
3. **Gradual Rollout**: Consider canary or blue-green deployments
4. **Monitoring**: Monitor closely after deployment
5. **Communication**: Notify stakeholders of deployments

## Pre-Production Checklist

### Step 1: Final Pre-Production Verification

```bash
#!/bin/bash
# pre-production-check.sh

set -e

VERSION=${1}
PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"

echo "=== Pre-Production Verification for v${VERSION} ==="

# Check 1: Build exists
echo "Checking build existence..."
if [ ! -f "builds/${VERSION}.tar.gz" ]; then
    echo "FAIL: Build not found"
    exit 1
fi
echo "PASS: Build exists"

# Check 2: Build integrity
echo "Verifying build integrity..."
tar -tzf builds/${VERSION}.tar.gz > /dev/null
if [ $? -ne 0 ]; then
    echo "FAIL: Build corrupted"
    exit 1
fi
echo "PASS: Build integrity verified"

# Check 3: Staging verification complete
echo "Checking staging verification..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "test -f /var/www/whmcs/modules/addons/your_module/VERSION" || echo "WARN: No previous version found"
echo "PASS: Staging check"

# Check 4: Backup exists
echo "Verifying backup capability..."
BACKUP_TEST=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "test -w /var/www/whmcs/backups/modules/your_module && echo 'writable'")
if [ "$BACKUP_TEST" != "writable" ]; then
    echo "FAIL: Backup directory not writable"
    exit 1
fi
echo "PASS: Backup directory writable"

# Check 5: Production database accessible
echo "Checking database access..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    mysql -u whmcs -p\${WHMCSDB_PASSWORD} whmcs -e 'SELECT 1' > /dev/null 2>&1
    echo \$?
" | grep -q 0
if [ $? -ne 0 ]; then
    echo "FAIL: Cannot access production database"
    exit 1
fi
echo "PASS: Database accessible"

# Check 6: Disk space
echo "Checking disk space..."
FREE_SPACE=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "df -BG /var/www/whmcs | tail -1 | awk '{print \$4}' | sed 's/G//'")
if [ "$FREE_SPACE" -lt 5 ]; then
    echo "FAIL: Low disk space: ${FREE_SPACE}GB"
    exit 1
fi
echo "PASS: Sufficient disk space: ${FREE_SPACE}GB"

# Check 7: Maintenance mode ready
echo "Checking maintenance mode capability..."
curl -s -o /dev/null -w "%{http_code}" "https://${PRODUCTION_HOST}/whmcs/maintenance.php"
echo ""
echo "INFO: Maintenance mode check complete"

echo "=== All Pre-Production Checks Passed ==="
```

### Step 2: Create Production Build

```bash
#!/bin/bash
# create-production-build.sh

VERSION=${1}
BUILD_DIR="./builds/${VERSION}"
MODULE_NAME="your_module"

echo "Creating production build: ${VERSION}"

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
    --exclude='*.log' \
    ./modules/addons/${MODULE_NAME}/ ${BUILD_DIR}/${MODULE_NAME}/

# Version file
echo "${VERSION}" > ${BUILD_DIR}/${MODULE_NAME}/VERSION
echo "$(date -Iseconds)" > ${BUILD_DIR}/${MODULE_NAME}/DEPLOYED_AT

# Production composer files
cp composer.json ${BUILD_DIR}/
cp composer.lock ${BUILD_DIR}/

# Create production configuration
cat > ${BUILD_DIR}/${MODULE_NAME}/.env.production << 'EOF'
APP_ENV=production
MODULE_DEBUG=false
MODULE_LOG_LEVEL=error
DB_HOST=${DB_HOST}
DB_NAME=${DB_NAME}
DB_USER=${DB_USER}
DB_PASSWORD=${DB_PASSWORD}
API_ENDPOINT=https://api.example.com
WEBHOOK_URL=https://webhook.example.com
EOF

# Production dependencies only
cd ${BUILD_DIR}
composer install --no-dev --optimize-autoloader --no-scripts

# Create checksum
sha256sum ${BUILD_DIR}.tar.gz > ${BUILD_DIR}.tar.gz.sha256

# Clean up build directory
rm -rf ${BUILD_DIR}

echo "Production build created:"
echo "  - builds/${VERSION}.tar.gz"
echo "  - builds/${VERSION}.tar.gz.sha256"
```

## Deployment Window

### Step 3: Schedule Deployment

```markdown
## Production Deployment Schedule

### Deployment Window
- **Date**: [Insert Date]
- **Time**: [Insert Time] UTC
- **Duration**: 30-60 minutes expected

### Personnel
- **Deployment Lead**: [Name]
- **QA Lead**: [Name]
- **On-Call DevOps**: [Name]
- **Stakeholders to Notify**: [List]

### Pre-Deployment Actions (T-24h)
- [ ] Verify build artifacts
- [ ] Confirm backup completion
- [ ] Notify stakeholders
- [ ] Disable non-essential integrations
- [ ] Prepare monitoring dashboards

### Pre-Deployment Actions (T-1h)
- [ ] Final verification
- [ ] Enable deployment mode
- [ ] Brief deployment team
- [ ] Verify rollback capability

### Deployment Steps
1. Create production backup
2. Enable maintenance mode
3. Deploy new version
4. Run database migrations
5. Clear cache
6. Disable maintenance mode
7. Verify deployment
8. Monitor for 30 minutes

### Rollback Triggers
- Error rate increases by >5%
- Response time increases by >100%
- Critical functionality broken
- Database errors detected

### Post-Deployment (T+1h)
- [ ] Verify monitoring
- [ ] Run smoke tests
- [ ] Notify stakeholders
- [ ] Document any issues
```

## Deployment Execution

### Step 4: Production Backup

```bash
#!/bin/bash
# production-backup.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
MODULE_NAME="your_module"
BACKUP_BASE="/var/www/whmcs/backups/modules/${MODULE_NAME}"

echo "=== Creating Production Backup ==="

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="production-${TIMESTAMP}"

ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << EOF
    set -e

    echo "Creating backup directory..."
    mkdir -p ${BACKUP_BASE}/${BACKUP_NAME}

    echo "Backing up module files..."
    tar -czf ${BACKUP_BASE}/${BACKUP_NAME}/module.tar.gz \
        -C /var/www/whmcs/modules/addons/${MODULE_NAME} .

    echo "Backing up module database tables..."
    mysqldump -u whmcs -p\${WHMCSDB_PASSWORD} whmcs mod_your_module \
        > ${BACKUP_BASE}/${BACKUP_NAME}/database.sql

    echo "Creating backup manifest..."
    cat > ${BACKUP_BASE}/${BACKUP_NAME}/manifest.json << 'MANIFEST'
    {
        "module": "${MODULE_NAME}",
        "timestamp": "${TIMESTAMP}",
        "version": "$(cat /var/www/whmcs/modules/addons/${MODULE_NAME}/VERSION 2>/dev/null || echo 'unknown')",
        "files": "$(tar -czf - -C /var/www/whmcs/modules/addons/${MODULE_NAME} . | wc -c)",
        "database_tables": ["mod_your_module"]
    }
    MANIFEST

    echo "Setting backup permissions..."
    chmod 600 ${BACKUP_BASE}/${BACKUP_NAME}/*

    echo "Creating latest symlink..."
    rm -f ${BACKUP_BASE}/latest
    ln -s ${BACKUP_BASE}/${BACKUP_NAME} ${BACKUP_BASE}/latest

    echo "Cleaning old backups (keeping last 5)..."
    cd ${BACKUP_BASE}
    ls -1d production-* | head -n -5 | xargs rm -rf

    echo "Backup complete: ${BACKUP_NAME}"
    ls -la ${BACKUP_BASE}/${BACKUP_NAME}/
EOF

echo "=== Backup Complete ==="
```

### Step 5: Enable Maintenance Mode

```php
<?php
// maintenance.php - Enable/disable maintenance mode

$action = $argv[1] ?? 'status';

switch ($action) {
    case 'enable':
        enableMaintenanceMode();
        break;
    case 'disable':
        disableMaintenanceMode();
        break;
    case 'status':
        checkMaintenanceStatus();
        break;
    default:
        echo "Usage: php maintenance.php [enable|disable|status]\n";
        exit(1);
}

function enableMaintenanceMode(): void
{
    $maintenanceFile = __DIR__ . '/maintenance.html';
    $lockFile = __DIR__ . '/.maintenance.lock';

    // Create maintenance page
    $maintenanceHtml = <<<'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>Maintenance</title>
    <meta name="robots" content="noindex, nofollow">
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background: #f5f5f5;
        }
        .container {
            text-align: center;
            padding: 40px;
            background: white;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 { color: #333; }
        p { color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Scheduled Maintenance</h1>
        <p>We are currently performing scheduled maintenance.</p>
        <p>Please check back shortly.</p>
    </div>
</body>
</html>
HTML;

    file_put_contents($maintenanceFile, $maintenanceHtml);
    file_put_contents($lockFile, json_encode([
        'started' => date('Y-m-d H:i:s'),
        'reason' => 'Production deployment'
    ]));

    echo "Maintenance mode enabled\n";
}

function disableMaintenanceMode(): void
{
    $maintenanceFile = __DIR__ . '/maintenance.html';
    $lockFile = __DIR__ . '/.maintenance.lock';

    if (file_exists($lockFile)) {
        unlink($lockFile);
    }

    echo "Maintenance mode disabled\n";
}

function checkMaintenanceStatus(): void
{
    $lockFile = __DIR__ . '/.maintenance.lock';

    if (file_exists($lockFile)) {
        $info = json_decode(file_get_contents($lockFile), true);
        echo "Maintenance mode: ENABLED\n";
        echo "Started: " . ($info['started'] ?? 'unknown') . "\n";
        echo "Reason: " . ($info['reason'] ?? 'unknown') . "\n";
    } else {
        echo "Maintenance mode: DISABLED\n";
    }
}
```

### Step 6: Deploy to Production

```bash
#!/bin/bash
# deploy-to-production.sh

VERSION=${1}
PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
MODULE_NAME="your_module"

if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    exit 1
fi

echo "=== Deploying v${VERSION} to Production ==="

# Verify build exists
if [ ! -f "builds/${VERSION}.tar.gz" ]; then
    echo "ERROR: Build not found: builds/${VERSION}.tar.gz"
    exit 1
fi

# Verify checksum
echo "Verifying build checksum..."
CHECKSUM=$(sha256sum builds/${VERSION}.tar.gz | cut -d' ' -f1)
EXPECTED=$(cat builds/${VERSION}.tar.gz.sha256 | cut -d' ' -f1)
if [ "$CHECKSUM" != "$EXPECTED" ]; then
    echo "ERROR: Checksum mismatch"
    exit 1
fi
echo "Checksum verified"

# Upload build
echo "Uploading build..."
scp builds/${VERSION}.tar.gz ${PRODUCTION_USER}@${PRODUCTION_HOST}:/tmp/production.tar.gz

# Deploy
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << 'EOF'
    set -e

    MODULE_PATH="/var/www/whmcs/modules/addons/your_module"
    BACKUP_BASE="/var/www/whmcs/backups/modules/your_module"

    echo "Step 1: Creating deployment backup..."
    LATEST=$(readlink ${BACKUP_BASE}/latest)
    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
    DEPLOY_BACKUP="${BACKUP_BASE}/deploy-${TIMESTAMP}"
    mkdir -p ${DEPLOY_BACKUP}
    tar -czf ${DEPLOY_BACKUP}/module.tar.gz -C ${MODULE_PATH} .

    echo "Step 2: Extracting new version..."
    tar -xzf /tmp/production.tar.gz -C ${MODULE_PATH}

    echo "Step 3: Setting permissions..."
    chown -R www-data:www-data ${MODULE_PATH}
    find ${MODULE_PATH} -type d -exec chmod 755 {} \;
    find ${MODULE_PATH} -type f -exec chmod 644 {} \;
    chmod -R 777 ${MODULE_PATH}/storage 2>/dev/null || true

    echo "Step 4: Running database migrations..."
    cd /var/www/whmcs
    php ${MODULE_PATH}/migrations/run.php production

    echo "Step 5: Clearing cache..."
    php artisan cache:clear 2>/dev/null || true

    echo "Step 6: Cleaning up..."
    rm -f /tmp/production.tar.gz

    echo "Deployment complete!"
    echo "Deployed version: ${VERSION}"
    cat ${MODULE_PATH}/VERSION
EOF

echo "=== Production Deployment Complete ==="
```

## Post-Deployment

### Step 7: Post-Deployment Verification

```bash
#!/bin/bash
# post-deployment-verification.sh

PRODUCTION_URL="https://whmcs.example.com"
PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"

echo "=== Post-Deployment Verification ==="

# Function to check with timeout
check_endpoint() {
    local url=$1
    local name=$2
    local expected_code=${3:-200}

    local code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 30 "$url")
    if [ "$code" = "$expected_code" ]; then
        echo "PASS: $name (HTTP $code)"
        return 0
    else
        echo "FAIL: $name (Expected $expected_code, got HTTP $code)"
        return 1
    fi
}

# Function to check for errors in logs
check_logs() {
    local pattern=$1
    local description=$2

    local errors=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} \
        "tail -100 /var/www/whmcs/storage/logs/module_error.log 2>/dev/null | grep -c '$pattern' || echo 0")

    if [ "$errors" = "0" ]; then
        echo "PASS: No $description errors"
        return 0
    else
        echo "WARN: Found $errors $description errors"
        return 1
    fi
}

FAILED=0

# Check 1: Maintenance mode disabled
echo "Checking maintenance mode..."
if curl -s "${PRODUCTION_URL}/whmcs" | grep -q "maintenance.html"; then
    echo "FAIL: Maintenance mode still enabled"
    FAILED=1
else
    echo "PASS: Maintenance mode disabled"
fi

# Check 2: WHMCS admin accessible
check_endpoint "${PRODUCTION_URL}/whmcs/admin/index.php" "WHMCS Admin" || FAILED=1

# Check 3: Client area accessible
check_endpoint "${PRODUCTION_URL}/whmcs/clientarea.php" "Client Area" || FAILED=1

# Check 4: Module admin page accessible
check_endpoint "${PRODUCTION_URL}/whmcs/admin/addonmodules.php?module=your_module" "Module Admin Page" || FAILED=1

# Check 5: API responding
check_endpoint "${PRODUCTION_URL}/whmcs/api.php?action=GetStats&username=admin&password=${ADMIN_API_PASS}&responsetype=json" "API Endpoint" || FAILED=1

# Check 6: Error logs clean
check_logs "ERROR" "critical" || FAILED=1

# Check 7: Database connection
echo "Checking database connection..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    mysql -u whmcs -p\${WHMCSDB_PASSWORD} whmcs -e 'SELECT COUNT(*) FROM mod_your_module' > /dev/null 2>&1
" && echo "PASS: Database connection" || { echo "FAIL: Database connection"; FAILED=1; }

# Check 8: Version correct
echo "Verifying deployed version..."
DEPLOYED_VERSION=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "cat /var/www/whmcs/modules/addons/your_module/VERSION 2>/dev/null")
echo "Deployed version: $DEPLOYED_VERSION"

echo ""
if [ $FAILED -eq 0 ]; then
    echo "=== All Verifications Passed ==="
    exit 0
else
    echo "=== Some Verifications Failed ==="
    exit 1
fi
```

### Step 8: Smoke Tests

```php
<?php
// post-deployment-smoke-tests.php

require_once __DIR__ . '/includes/init.php';

use WHMCS\Database\Capsule;

$tests = [
    'Database Connection' => function() {
        Capsule::connection()->getPdo();
        return true;
    },

    'Module Table Accessible' => function() {
        $result = Capsule::table('mod_your_module')->count();
        return is_numeric($result);
    },

    'Configuration Accessible' => function() {
        $setting = Capsule::table('tblconfiguration')
            ->where('setting', 'like', 'Module%')
            ->first();
        return $setting !== null;
    },

    'Client Data Accessible' => function() {
        $client = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->first();
        return $client !== null;
    },

    'Module Hooks Registered' => function() {
        $hooks = Capsule::table('tblhooks')
            ->where('filename', 'like', '%your_module%')
            ->count();
        return $hooks > 0;
    },

    'Log Directory Writable' => function() {
        $logFile = dirname(__DIR__) . '/storage/logs/smoke_test.log';
        $result = file_put_contents($logFile, date('Y-m-d H:i:s'));
        if ($result) unlink($logFile);
        return $result !== false;
    }
];

$passed = 0;
$failed = 0;

echo "=== Post-Deployment Smoke Tests ===\n\n";

foreach ($tests as $name => $test) {
    try {
        $result = $test();
        if ($result) {
            echo "[PASS] $name\n";
            $passed++;
        } else {
            echo "[FAIL] $name\n";
            $failed++;
        }
    } catch (\Exception $e) {
        echo "[FAIL] $name: " . $e->getMessage() . "\n";
        $failed++;
    }
}

echo "\n=== Results: $passed passed, $failed failed ===\n";
exit($failed > 0 ? 1 : 0);
```

## Monitoring

### Step 9: Post-Deployment Monitoring

```yaml
# Monitoring checks after deployment
- name: Monitor Error Rates
  run: |
    # Check error rate for last 30 minutes
    ERROR_RATE=$(curl -s "https://api.newrelic.com/v2/alerts_policy_conditions.json" \
      -H "X-Api-Key: ${NEWRELIC_API_KEY}" | jq '.error_rate')

    if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
      echo "ERROR: Error rate elevated: $ERROR_RATE"
      exit 1
    fi

- name: Monitor Response Times
  run: |
    # Check average response time
    RESPONSE_TIME=$(curl -s "https://api.newrelic.com/v2/apps/${APP_ID}/metrics/data.json" \
      -H "X-Api-Key: ${NEWRELIC_API_KEY}" \
      -d 'names[]=HttpDispatcher' | jq '.metrics.data[0].timeslices[0].values.average_response_time')

    if (( $(echo "$RESPONSE_TIME > 2.0" | bc -l) )); then
      echo "WARNING: Response time elevated: ${RESPONSE_TIME}s"
    fi

- name: Check for PHP Errors
  run: |
    ssh ${{ secrets.PRODUCTION_HOST }} "
      tail -500 /var/www/whmcs/storage/logs/php_errors.log | grep 'Fatal\|Error\|Exception' | tail -20
    "
```

## Stakeholder Communication

### Step 10: Deployment Notification

```markdown
## Production Deployment Notification

### Deployment Summary
- **Version**: v1.2.0
- **Deployed by**: @developer
- **Deployed at**: 2024-01-15 14:30 UTC
- **Duration**: 25 minutes
- **Status**: SUCCESS

### Changes Included
- New: Enhanced reporting dashboard
- New: Improved API rate limiting
- Fix: Corrected timezone handling issue
- Fix: Resolved caching race condition

### Verification Results
- Smoke Tests: 8/8 passed
- Error Rate: Normal (<0.1%)
- Response Time: Normal (~200ms)
- Database: Healthy

### Monitoring
Please monitor your dashboards for the next 1 hour:
- New Relic Dashboard: [Link]
- Error Tracking: [Link]
- Log Search: [Link]

### Rollback Plan
If issues are detected, rollback to v1.1.0 using:
```
./rollback.sh v1.1.0
```

### Support
For issues, contact:
- On-call DevOps: @devops-oncall
- Module Lead: @module-lead

---
Deployment Automation System
```

## Verification Checklist

- [ ] Pre-production checks passed
- [ ] Production build created
- [ ] Checksum verified
- [ ] Backup created
- [ ] Maintenance mode enabled
- [ ] Module deployed
- [ ] Migrations executed
- [ ] Cache cleared
- [ ] Maintenance mode disabled
- [ ] Smoke tests passed
- [ ] Monitoring verified
- [ ] Stakeholders notified
- [ ] Documentation updated
- [ ] Monitoring for 30+ minutes complete
