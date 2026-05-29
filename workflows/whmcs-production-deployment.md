# WHMCS Production Deployment Workflow

## Overview
This workflow guides you through safely deploying WHMCS modules to production.

## Prerequisites
- Staging deployment complete
- Production server access
- Backup system
- Monitoring

## Step-by-Step Guide

### Step 1: Pre-Deployment Checklist
```markdown
# Pre-Deployment Checklist

## Code Review
- [ ] All changes reviewed
- [ ] Code follows standards
- [ ] Security review complete
- [ ] Performance reviewed

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Staging tests pass
- [ ] No critical bugs

## Documentation
- [ ] Changelog updated
- [ ] README updated
- [ ] API docs updated
```

### Step 2: Create Production Deployment Script
```bash
#!/bin/bash
# deploy-production.sh

set -e

MODULE_NAME="yourmodule"
MODULE_DIR="/var/www/whmcs/html/modules/addons/$MODULE_NAME"
BACKUP_DIR="/var/www/whmcs/backups/modules/$MODULE_NAME"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
GIT_REPO="https://github.com/your-org/whmcs-module.git"
VERSION="${1:-latest}"

echo "=== WHMCS Module Deployment ==="
echo "Module: $MODULE_NAME"
echo "Version: $VERSION"
echo "Timestamp: $TIMESTAMP"

# Step 1: Create backup
echo "[1/6] Creating backup..."
mkdir -p "$BACKUP_DIR"
if [ -d "$MODULE_DIR" ]; then
    cp -r "$MODULE_DIR" "$BACKUP_DIR/backup_$TIMESTAMP"
    echo "Backup created at $BACKUP_DIR/backup_$TIMESTAMP"
fi

# Step 2: Pull latest code
echo "[2/6] Pulling latest code..."
if [ -d "$MODULE_DIR/.git" ]; then
    cd "$MODULE_DIR"
    git pull origin main
else
    rm -rf "$MODULE_DIR"
    git clone -b main "$GIT_REPO" "$MODULE_DIR"
fi

# Step 3: Install dependencies
echo "[3/6] Installing dependencies..."
cd "$MODULE_DIR"
composer install --no-dev --optimize-autoloader

# Step 4: Set permissions
echo "[4/6] Setting permissions..."
chmod -R 755 "$MODULE_DIR"
find "$MODULE_DIR" -type d -name storage -exec chmod -R 775 {} \; 2>/dev/null || true

# Step 5: Clear cache
echo "[5/6] Clearing cache..."
rm -rf /var/www/whmcs/admin/downloads/cache/* 2>/dev/null || true
rm -rf /var/www/whmcs/templates_c/* 2>/dev/null || true

# Step 6: Verify deployment
echo "[6/6] Verifying deployment..."
if [ -f "$MODULE_DIR/yourmodule.php" ]; then
    echo "Module file exists: OK"
else
    echo "Module file missing: FAILED"
    exit 1
fi

echo "=== Deployment Complete ==="
echo "Module deployed at: $MODULE_DIR"
echo "Backup at: $BACKUP_DIR/backup_$TIMESTAMP"
```

### Step 3: Run Pre-Deployment Checks
```bash
#!/bin/bash
# pre-deployment-check.sh

echo "Running pre-deployment checks..."

# Check disk space
DISK_USAGE=$(df -h /var/www | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 80 ]; then
    echo "WARNING: Disk usage is ${DISK_USAGE}%"
fi

# Check database connectivity
mysql -h localhost -u whmcs_user -pwhmcs_password -e "SELECT 1" > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "ERROR: Cannot connect to database"
    exit 1
fi

# Check for existing module
if [ -d "/var/www/whmcs/html/modules/addons/yourmodule" ]; then
    echo "Module exists - will update"
else
    echo "New module installation"
fi

# Check for pending migrations
cd /var/www/whmcs/html/modules/addons/yourmodule
php artisan migrate:status 2>/dev/null || echo "No migrations"

echo "Pre-deployment checks complete"
```

### Step 4: Execute Deployment
```bash
# Make scripts executable
chmod +x deploy-production.sh pre-deployment-check.sh

# Run pre-deployment checks
./pre-deployment-check.sh

# Deploy (with specific version)
./deploy-production.sh v2.0.0

# Or deploy latest from main
./deploy-production.sh latest
```

### Step 5: Post-Deployment Verification
```bash
#!/bin/bash
# post-deployment-check.sh

MODULE_NAME="yourmodule"
MODULE_DIR="/var/www/whmcs/html/modules/addons/$MODULE_NAME"

echo "=== Post-Deployment Verification ==="

# Check file exists
if [ -f "$MODULE_DIR/$MODULE_NAME.php" ]; then
    echo "[OK] Module file exists"
else
    echo "[FAIL] Module file missing"
    exit 1
fi

# Check module version
VERSION=$(grep -oP "VERSION\s*=\s*['\"]?\K[^'\"]+" "$MODULE_DIR/$MODULE_NAME.php" || echo "unknown")
echo "[OK] Module version: $VERSION"

# Check permissions
PERMS=$(stat -c %a "$MODULE_DIR/$MODULE_NAME.php")
echo "[OK] File permissions: $PERMS"

# Test module activation
curl -s -o /dev/null -w "%{http_code}" \
    "https://your-whmcs.com/admin/modules/addons/$MODULE_NAME/activate.php"

# Check error logs
ERRORS=$(tail -100 /var/www/whmcs/admin/logs/*.log | grep -i "$MODULE_NAME" | tail -5)
if [ -n "$ERRORS" ]; then
    echo "[WARN] Errors found in logs:"
    echo "$ERRORS"
fi

echo "=== Verification Complete ==="
```

### Step 6: Activate in WHMCS Admin
1. Log in to WHMCS Admin
2. Go to Configuration > System Settings > Module Settings
3. Activate module
4. Configure settings
5. Verify functionality

## Production Deployment Checklist

### Pre-Deployment
- [ ] Code reviewed and approved
- [ ] All tests passing
- [ ] Backup created
- [ ] Rollback plan ready
- [ ] Communication sent

### Deployment
- [ ] Scripts executed
- [ ] No errors
- [ ] Permissions correct
- [ ] Cache cleared

### Post-Deployment
- [ ] Module activates
- [ ] Configuration works
- [ ] Features functional
- [ ] No errors in logs
- [ ] Performance acceptable

### Monitoring
- [ ] Error rates normal
- [ ] Response times acceptable
- [ ] User reports monitored
