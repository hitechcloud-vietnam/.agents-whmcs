# WHMCS Staging Deployment Workflow

## Overview
This workflow guides you through deploying WHMCS modules to a staging environment for pre-production testing.

## Prerequisites
- WHMCS installation on staging server
- Deployment script
- Test suite

## Step-by-Step Guide

### Step 1: Prepare Staging Environment
```bash
# SSH to staging server
ssh staging@your-staging-server.com

# Create staging directory
sudo mkdir -p /var/www/whmcs-staging
sudo chown -R staging:www-data /var/www/whmcs-staging

# Clone repository
cd /var/www/whmcs-staging
git clone https://github.com/your-org/whmcs-module.git module
```

### Step 2: Configure Staging Database
```bash
mysql -u root -p
CREATE DATABASE whmcs_staging;
CREATE USER 'whmcs_staging'@'localhost' IDENTIFIED BY 'staging_password';
GRANT ALL PRIVILEGES ON whmcs_staging.* TO 'whmcs_staging'@'localhost';
FLUSH PRIVILEGES;
```

### Step 3: Deploy Module
```bash
# Using deployment script
#!/bin/bash
# deploy-staging.sh

MODULE_DIR="/var/www/whmcs-staging/html/modules/addons/yourmodule"
GIT_REPO="https://github.com/your-org/whmcs-module.git"
BRANCH="${1:-main}"

echo "Deploying from $GIT_REPO (branch: $BRANCH)"

# Clone or pull
if [ -d "$MODULE_DIR/.git" ]; then
    cd "$MODULE_DIR"
    git pull origin "$BRANCH"
else
    rm -rf "$MODULE_DIR"
    git clone -b "$BRANCH" "$GIT_REPO" "$MODULE_DIR"
fi

# Install dependencies
cd "$MODULE_DIR"
composer install --no-dev --optimize-autoloader

# Set permissions
chmod -R 755 "$MODULE_DIR"
chmod -R 775 "$MODULE_DIR/storage" 2>/dev/null || true

# Clear cache
rm -rf /var/www/whmcs-staging/html/admin/downloads/cache/*

echo "Deployment complete"
```

### Step 4: Run Deployment
```bash
# Make script executable
chmod +x deploy-staging.sh

# Run deployment
./deploy-staging.sh develop
```

### Step 5: Verify Deployment
```bash
# Check module exists
ls -la /var/www/whmcs-staging/html/modules/addons/yourmodule/

# Check for errors
tail -50 /var/www/whmcs-staging/html/admin/logs/module.log

# Verify file integrity
cd /var/www/whmcs-staging/html/modules/addons/yourmodule
git status
git log --oneline -5
```

### Step 6: Activate Module in WHMCS Admin
1. Log in to WHMCS Admin on staging
2. Navigate to Configuration > System Settings > Module Settings
3. Find and activate your module
4. Configure settings
5. Run test scenarios

### Step 7: Automated Verification
```bash
#!/bin/bash
# verify-staging.sh

STAGING_URL="https://staging.yourdomain.com"

echo "Running verification tests..."

# Test module activation
curl -s -o /dev/null -w "%{http_code}" \
    "$STAGING_URL/admin/modules/addons/yourmodule/activate.php"

# Run PHPUnit
cd /var/www/whmcs-staging/html/modules/addons/yourmodule
./vendor/bin/phpunit --testsuite Staging

echo "Verification complete"
```

## Staging Deployment Checklist

### Environment
- [ ] Staging server accessible
- [ ] WHMCS installed on staging
- [ ] Database configured
- [ ] SSL certificate valid

### Deployment
- [ ] Module files deployed
- [ ] Dependencies installed
- [ ] Permissions set
- [ ] Cache cleared

### Verification
- [ ] Module activates successfully
- [ ] Configuration saves
- [ ] Test suite passes
- [ ] No errors in logs

### Testing
- [ ] Smoke tests pass
- [ ] Integration tests pass
- [ ] Performance acceptable
- [ ] Security checks pass
