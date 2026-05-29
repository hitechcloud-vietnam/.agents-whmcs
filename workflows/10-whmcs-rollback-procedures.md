# WHMCS Rollback Procedures

## Overview
This workflow defines the procedures for safely rolling back WHMCS module deployments when issues are detected in production.

## Rollback Triggers

### Automatic Triggers
- Error rate increases by more than 5%
- Response time increases by more than 100%
- Critical functionality broken
- Database errors detected
- Payment processing failures

### Manual Triggers
- QA testing reveals critical issues
- Stakeholder decision to revert
- Security vulnerability discovered

## Rollback Decision Matrix

| Severity | Issue | Rollback Required |
|----------|-------|-------------------|
| Critical | Payment processing broken | Immediate |
| Critical | Client data exposure | Immediate |
| High | Module causes 500 errors | Within 1 hour |
| High | Core WHMCS features broken | Within 1 hour |
| Medium | Non-critical features broken | Evaluate |
| Low | Cosmetic issues | No |

## Pre-Rollback Checklist

### Step 1: Assess the Situation

```bash
#!/bin/bash
# assess-rollback-need.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"

echo "=== Rollback Need Assessment ==="

# Check 1: Current error rate
echo "Checking error rate..."
ERROR_RATE=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    tail -500 /var/www/whmcs/storage/logs/php_errors.log | grep -c 'Error\|Exception' || echo 0
")
echo "Errors in last 500 lines: $ERROR_RATE"

# Check 2: Recent changes
echo "Checking recent changes..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    cat /var/www/whmcs/modules/addons/your_module/DEPLOYED_AT 2>/dev/null
    cat /var/www/whmcs/modules/addons/your_module/VERSION 2>/dev/null
"

# Check 3: Database status
echo "Checking database..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    mysql -u whmcs -p\${WHMCSDB_PASSWORD} whmcs -e 'SELECT COUNT(*) FROM mod_your_module' 2>/dev/null | tail -1
"

# Check 4: Available backups
echo "Available backups:"
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    ls -la /var/www/whmcs/backups/modules/your_module/production-* 2>/dev/null | tail -5
"

# Check 5: Last successful version
echo "Last successful backup:"
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    readlink /var/www/whmcs/backups/modules/your_module/latest
"
```

### Step 2: Get Approval

```markdown
## Rollback Approval Request

### Incident Report
- **Time Detected**: [Timestamp]
- **Detected By**: [Name]
- **Severity**: [Critical/High/Medium/Low]

### Issue Description
[Detailed description of the issue]

### Impact Assessment
- **Users Affected**: [Number/Percentage]
- **Business Impact**: [Description]
- **Financial Impact**: [If applicable]

### Rollback Plan
- **Target Version**: [Previous version to restore]
- **Expected Duration**: [Time to complete rollback]
- **Risk Level**: [Low/Medium/High]

### Alternative Options Considered
1. Hotfix deployment
2. Feature flag disable
3. Database rollback only

### Recommendation
[Rollback recommended / Alternative approach]

### Approval Required
- [ ] DevOps Lead
- [ ] Product Owner
- [ ] CTO (if Critical)
```

## Rollback Execution

### Step 3: Prepare Rollback

```bash
#!/bin/bash
# prepare-rollback.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
MODULE_NAME="your_module"
TARGET_VERSION=${1}

if [ -z "$TARGET_VERSION" ]; then
    echo "Usage: $0 <version|latest>"
    exit 1
fi

echo "=== Preparing Rollback to ${TARGET_VERSION} ==="

# Determine backup to use
if [ "$TARGET_VERSION" = "latest" ]; then
    BACKUP_PATH=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
        readlink /var/www/whmcs/backups/modules/${MODULE_NAME}/latest
    ")
else
    # Find backup for version
    BACKUP_PATH=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
        ls /var/www/whmcs/backups/modules/${MODULE_NAME}/ | while read dir; do
            if tar -xzf /var/www/whmcs/backups/modules/${MODULE_NAME}/\$dir/module.tar.gz -O VERSION 2>/dev/null | grep -q ${TARGET_VERSION}; then
                echo \$dir
                break
            fi
        done
    ")
fi

if [ -z "$BACKUP_PATH" ]; then
    echo "ERROR: Backup not found for version ${TARGET_VERSION}"
    exit 1
fi

echo "Using backup: $BACKUP_PATH"

# Verify backup integrity
echo "Verifying backup integrity..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    tar -tzf /var/www/whmcs/backups/modules/${MODULE_NAME}/${BACKUP_PATH}/module.tar.gz > /dev/null
    if [ \$? -eq 0 ]; then
        echo 'Backup integrity: OK'
    else
        echo 'Backup integrity: FAILED'
        exit 1
    fi
"

# Create pre-rollback backup
echo "Creating pre-rollback backup..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    TIMESTAMP=\$(date +%Y%m%d_%H%M%S)
    PRE_ROLLBACK_DIR=/var/www/whmcs/backups/modules/${MODULE_NAME}/pre-rollback-\${TIMESTAMP}
    mkdir -p \${PRE_ROLLBACK_DIR}
    tar -czf \${PRE_ROLLBACK_DIR}/module.tar.gz -C /var/www/whmcs/modules/addons/${MODULE_NAME} .
    echo 'Pre-rollback backup created: '\${PRE_ROLLBACK_DIR}
"

echo "=== Rollback Preparation Complete ==="
```

### Step 4: Enable Maintenance Mode

```bash
#!/bin/bash
# enable-maintenance.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"

echo "Enabling maintenance mode..."

ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << 'EOF'
    # Create maintenance mode indicator
    touch /var/www/whmcs/.maintenance.lock
    echo '{"reason": "Rollback in progress", "started": "'$(date -Iseconds)'"}' > /var/www/whmcs/.maintenance.lock

    # Optionally enable WHMCS maintenance (if available)
    # php artisan down

    echo "Maintenance mode enabled"
    echo "Reason: Rollback in progress"
    echo "Time: $(date)"
EOF
```

### Step 5: Execute Rollback

```bash
#!/bin/bash
# execute-rollback.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
MODULE_NAME="your_module"
TARGET_VERSION=${1}
BACKUP_PATH=${2}

if [ -z "$TARGET_VERSION" ] || [ -z "$BACKUP_PATH" ]; then
    echo "Usage: $0 <target_version> <backup_path>"
    exit 1
fi

echo "=== Executing Rollback to ${TARGET_VERSION} ==="

ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << 'EOF'
    set -e

    MODULE_PATH="/var/www/whmcs/modules/addons/your_module"
    BACKUP_BASE="/var/www/whmcs/backups/modules/your_module"

    echo "Step 1: Backing up current state..."
    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
    CURRENT_BACKUP="${BACKUP_BASE}/current-$(cat ${MODULE_PATH}/VERSION 2>/dev/null)-${TIMESTAMP}"
    mkdir -p ${CURRENT_BACKUP}
    tar -czf ${CURRENT_BACKUP}/module.tar.gz -C ${MODULE_PATH} .

    # Backup database state
    mysqldump -u whmcs -p${WHMCSDB_PASSWORD} whmcs mod_your_module \
        > ${CURRENT_BACKUP}/database.sql 2>/dev/null || true

    echo "Current state backed up to: ${CURRENT_BACKUP}"

    echo "Step 2: Rolling back database (if needed)..."
    # Check if rollback SQL exists
    if [ -f "${BACKUP_BASE}/${BACKUP_PATH}/rollback.sql" ]; then
        mysql -u whmcs -p${WHMCSDB_PASSWORD} whmcs < ${BACKUP_BASE}/${BACKUP_PATH}/rollback.sql 2>/dev/null || true
        echo "Database rolled back"
    else
        echo "No database rollback needed"
    fi

    echo "Step 3: Restoring module files..."
    # Clear current files
    rm -rf ${MODULE_PATH}/*
    # Extract backup
    tar -xzf ${BACKUP_BASE}/${BACKUP_PATH}/module.tar.gz -C ${MODULE_PATH}

    echo "Step 4: Setting permissions..."
    chown -R www-data:www-data ${MODULE_PATH}
    find ${MODULE_PATH} -type d -exec chmod 755 {} \;
    find ${MODULE_PATH} -type f -exec chmod 644 {} \;
    chmod -R 777 ${MODULE_PATH}/storage 2>/dev/null || true

    echo "Step 5: Clearing cache..."
    cd /var/www/whmcs
    php artisan cache:clear 2>/dev/null || true

    echo "Step 6: Verifying rollback..."
    CURRENT_VERSION=$(cat ${MODULE_PATH}/VERSION 2>/dev/null || echo 'unknown')
    echo "Module now at version: ${CURRENT_VERSION}"

    echo "=== Rollback Complete ==="
EOF

echo "Rollback completed successfully"
```

### Step 6: Disable Maintenance Mode

```bash
#!/bin/bash
# disable-maintenance.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"

echo "Disabling maintenance mode..."

ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << 'EOF'
    # Remove maintenance lock
    rm -f /var/www/whmcs/.maintenance.lock

    # Re-enable WHMCS (if using artisan)
    # cd /var/www/whmcs && php artisan up

    echo "Maintenance mode disabled"
    echo "Time: $(date)"
EOF
```

## Post-Rollback Verification

### Step 7: Verify Rollback

```bash
#!/bin/bash
# verify-rollback.sh

PRODUCTION_URL="https://whmcs.example.com"
PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
EXPECTED_VERSION=${1}

echo "=== Post-Rollback Verification ==="

FAILED=0

# Check 1: Version correct
echo "Checking version..."
DEPLOYED_VERSION=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "cat /var/www/whmcs/modules/addons/your_module/VERSION 2>/dev/null")
if [ "$DEPLOYED_VERSION" = "$EXPECTED_VERSION" ]; then
    echo "PASS: Version correct ($EXPECTED_VERSION)"
else
    echo "FAIL: Version mismatch (expected: $EXPECTED_VERSION, got: $DEPLOYED_VERSION)"
    FAILED=1
fi

# Check 2: Maintenance mode disabled
echo "Checking maintenance mode..."
if ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "test -f /var/www/whmcs/.maintenance.lock"; then
    echo "FAIL: Maintenance mode still enabled"
    FAILED=1
else
    echo "PASS: Maintenance mode disabled"
fi

# Check 3: Module accessible
echo "Checking module..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 30 "${PRODUCTION_URL}/whmcs/admin/addonmodules.php?module=your_module")
if [ "$HTTP_CODE" = "200" ]; then
    echo "PASS: Module admin accessible"
else
    echo "FAIL: Module admin not accessible (HTTP $HTTP_CODE)"
    FAILED=1
fi

# Check 4: Client area working
echo "Checking client area..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 30 "${PRODUCTION_URL}/whmcs/clientarea.php")
if [ "$HTTP_CODE" = "200" ]; then
    echo "PASS: Client area accessible"
else
    echo "FAIL: Client area not accessible (HTTP $HTTP_CODE)"
    FAILED=1
fi

# Check 5: No recent errors
echo "Checking error logs..."
ERRORS=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    tail -100 /var/www/whmcs/storage/logs/module_error.log 2>/dev/null | grep -c 'timestamp' || echo 0
")
if [ "$ERRORS" -lt 5 ]; then
    echo "PASS: Error count acceptable"
else
    echo "WARN: Multiple errors in logs"
fi

# Check 6: Database integrity
echo "Checking database..."
ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
    mysql -u whmcs -p\${WHMCSDB_PASSWORD} whmcs -e 'SELECT COUNT(*) FROM mod_your_module' > /dev/null 2>&1
" && echo "PASS: Database accessible" || { echo "FAIL: Database error"; FAILED=1; }

echo ""
if [ $FAILED -eq 0 ]; then
    echo "=== Rollback Verification Passed ==="
    exit 0
else
    echo "=== Rollback Verification Failed ==="
    exit 1
fi
```

### Step 8: Extended Monitoring

```bash
#!/bin/bash
# extended-monitoring.sh

PRODUCTION_URL="https://whmcs.example.com"
PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
MONITOR_DURATION=3600  # 1 hour

echo "=== Starting Extended Monitoring ==="
echo "Duration: $((MONITOR_DURATION/60)) minutes"

START_TIME=$(date +%s)
END_TIME=$((START_TIME + MONITOR_DURATION))

while [ $(date +%s) -lt $END_TIME ]; do
    ELAPSED=$(($(date +%s) - START_TIME))
    REMAINING=$((END_TIME - $(date +%s)))

    # Check error rate
    ERRORS=$(ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "
        tail -200 /var/www/whmcs/storage/logs/module_error.log 2>/dev/null | grep -c 'Error\|Exception' || echo 0
    ")

    # Check response time
    RESPONSE=$(curl -s -o /dev/null -w "%{time_total}" --max-time 10 "${PRODUCTION_URL}/whmcs/clientarea.php")

    # Check HTTP status
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "${PRODUCTION_URL}/whmcs/")

    echo "[$(date '+%H:%M:%S')] Elapsed: ${ELAPSED}s | Errors: $ERRORS | Response: ${RESPONSE}s | HTTP: $HTTP_CODE"

    if [ "$ERRORS" -gt 10 ] || [ "$(echo "$RESPONSE > 5" | bc)" = "1" ] || [ "$HTTP_CODE" != "200" ]; then
        echo "ALERT: Issues detected during monitoring!"
        # Send alert
        break
    fi

    sleep 60  # Check every minute
done

echo "=== Extended Monitoring Complete ==="
```

## Communication

### Step 9: Rollback Notification

```markdown
## Rollback Notification

### Summary
- **Action**: Module rolled back
- **From Version**: v1.2.0
- **To Version**: v1.1.0
- **Duration**: 15 minutes
- **Status**: COMPLETED

### Reason for Rollback
[Brief explanation of why rollback was performed]

### Timeline
- 14:30 - Issue detected
- 14:32 - Assessment started
- 14:35 - Rollback approved
- 14:38 - Maintenance mode enabled
- 14:40 - Rollback executed
- 14:45 - Verification complete
- 14:48 - Maintenance mode disabled

### Impact
- Downtime: ~8 minutes
- Users Affected: All users
- Data Loss: None

### Current Status
- Previous version restored
- All systems operational
- Monitoring active

### Next Steps
1. Investigate root cause of v1.2.0 issues
2. Plan hotfix for v1.2.1
3. Schedule new deployment window

### Contact
On-call DevOps: @devops-oncall
```

## Database Rollback (If Required)

### Step 10: Database Rollback Procedures

```sql
-- database-rollback.sql
-- Run this ONLY if database changes need to be reverted

-- Step 1: Create backup of current database state
CREATE TABLE IF NOT EXISTS mod_your_module_backup AS SELECT * FROM mod_your_module;

-- Step 2: Restore from pre-deployment backup
-- (This would restore data from the backup taken before deployment)

-- Step 3: Verify restoration
SELECT COUNT(*) FROM mod_your_module;
SELECT MAX(updated_at) FROM mod_your_module;

-- Step 4: Clean up if successful
-- DROP TABLE IF EXISTS mod_your_module_backup;
```

```bash
#!/bin/bash
# database-rollback.sh

PRODUCTION_HOST="whmcs.example.com"
PRODUCTION_USER="deploy"
BACKUP_NAME=${1}

if [ -z "$BACKUP_NAME" ]; then
    echo "Usage: $0 <backup_name>"
    exit 1
fi

ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} << 'EOF'
    set -e

    echo "=== Database Rollback ==="

    # Create current state backup
    echo "Backing up current database state..."
    mysqldump -u whmcs -p${WHMCSDB_PASSWORD} whmcs mod_your_module \
        > /var/www/whmcs/backups/modules/your_module/pre-db-rollback-$(date +%Y%m%d-%H%M%S).sql

    # Restore from backup
    echo "Restoring database from: ${BACKUP_NAME}"
    mysql -u whmcs -p${WHMCSDB_PASSWORD} whmcs < /var/www/whmcs/backups/modules/your_module/${BACKUP_NAME}/database.sql

    # Verify
    echo "Verifying restoration..."
    ROW_COUNT=$(mysql -u whmcs -p${WHMCSDB_PASSWORD} whmcs -se "SELECT COUNT(*) FROM mod_your_module")
    echo "Rows in table: $ROW_COUNT"

    echo "=== Database Rollback Complete ==="
EOF
```

## Verification Checklist

- [ ] Issue assessment completed
- [ ] Rollback approval obtained
- [ ] Rollback preparation done
- [ ] Pre-rollback backup created
- [ ] Maintenance mode enabled
- [ ] Module rollback executed
- [ ] Database rollback executed (if needed)
- [ ] Permissions corrected
- [ ] Cache cleared
- [ ] Maintenance mode disabled
- [ ] Version verified correct
- [ ] Functionality verified working
- [ ] Error logs checked
- [ ] Extended monitoring started
- [ ] Stakeholders notified
- [ ] Post-incident report scheduled

## Rollback Prevention Best Practices

1. **Comprehensive Testing**: Always test thoroughly in staging
2. **Feature Flags**: Use feature flags for risky features
3. **Canary Deployments**: Deploy to small percentage first
4. **Blue-Green**: Maintain two environments
5. **Database Migrations**: Make migrations backward compatible
6. **Regular Backups**: Schedule automated backups
7. **Monitoring**: Set up comprehensive alerting
8. **Documentation**: Document rollback procedures
