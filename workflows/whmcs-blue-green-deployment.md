# WHMCS Blue-Green Deployment Workflow

## Overview
This workflow guides you through implementing blue-green deployment strategy for WHMCS modules.

## Prerequisites
- Two identical environments
- Load balancer
- Deployment scripts
- Monitoring

## Step-by-Step Guide

### Step 1: Set Up Environments
```bash
# Environment directories
BLUE_DIR="/var/www/whmcs-blue/html/modules/addons/yourmodule"
GREEN_DIR="/var/www/whmcs-green/html/modules/addons/yourmodule"

# Initial setup - clone both environments
git clone https://github.com/your-org/whmcs-module.git "$BLUE_DIR"
git clone https://github.com/your-org/whmcs-module.git "$GREEN_DIR"
```

### Step 2: Create Deployment Scripts
```bash
#!/bin/bash
# blue-green-deploy.sh

set -e

MODULE_NAME="yourmodule"
ACTIVE_COLOR="${1:-blue}"
DEPLOY_COLOR=$([ "$ACTIVE_COLOR" = "blue" ] && echo "green" || echo "blue")

DEPLOY_DIR="/var/www/whmcs-$DEPLOY_COLOR/html/modules/addons/$MODULE_NAME"
GIT_REPO="https://github.com/your-org/whmcs-module.git"

echo "=== Blue-Green Deployment ==="
echo "Active: $ACTIVE_COLOR"
echo "Deploying to: $DEPLOY_COLOR"

# Deploy to inactive environment
echo "[1/5] Cloning latest code..."
if [ -d "$DEPLOY_DIR/.git" ]; then
    cd "$DEPLOY_DIR"
    git pull origin main
else
    rm -rf "$DEPLOY_DIR"
    git clone "$GIT_REPO" "$DEPLOY_DIR"
fi

# Install dependencies
echo "[2/5] Installing dependencies..."
cd "$DEPLOY_DIR"
composer install --no-dev --optimize-autoloader

# Set permissions
echo "[3/5] Setting permissions..."
chmod -R 755 "$DEPLOY_DIR"

# Activate on deploy environment
echo "[4/5] Activating on $DEPLOY_COLOR environment..."
curl -s "https://whmcs-$DEPLOY_COLOR.yourdomain.com/admin/modules/addons/$MODULE_NAME/activate.php"

# Run smoke tests
echo "[5/5] Running smoke tests..."
./smoke-tests.sh "https://whmcs-$DEPLOY_COLOR.yourdomain.com"

# Switch traffic
echo "Ready to switch traffic to $DEPLOY_COLOR"
```

### Step 3: Switch Traffic Script
```bash
#!/bin/bash
# switch-traffic.sh

set -e

TARGET="${1:-green}"  # green or blue

echo "=== Switching Traffic to $TARGET ==="

# Update load balancer configuration
# This depends on your load balancer (nginx, haproxy, etc.)

case "$TARGET" in
    green)
        # Switch nginx upstream
        ln -sf /etc/nginx/sites-available/whmcs-green /etc/nginx/sites-enabled/whmcs
        nginx -t && nginx -s reload
        ;;
    blue)
        ln -sf /etc/nginx/sites-available/whmcs-blue /etc/nginx/sites-enabled/whmcs
        nginx -t && nginx -s reload
        ;;
esac

echo "Traffic switched to $TARGET"

# Verify
curl -s -o /dev/null -w "%{http_code}" "https://your-whmcs.com/health"
```

### Step 4: Rollback Script
```bash
#!/bin/bash
# blue-green-rollback.sh

CURRENT="${1:-green}"
SWITCH_TO=$([ "$CURRENT" = "green" ] && echo "blue" || echo "green")

echo "=== Rolling Back to $SWITCH_TO ==="

# Switch traffic back
./switch-traffic.sh "$SWITCH_TO"

echo "Rolled back to $SWITCH_TO"
```

### Step 5: Execute Deployment
```bash
# Deploy new version to green (if blue is active)
./blue-green-deploy.sh blue

# Run tests on green
./smoke-tests.sh "https://whmcs-green.yourdomain.com"

# Switch traffic to green
./switch-traffic.sh green

# Verify
curl -s "https://your-whmcs.com/health"
```

## Blue-Green Deployment Checklist

### Preparation
- [ ] Both environments configured
- [ ] DNS ready
- [ ] Load balancer configured
- [ ] Monitoring set up

### Deployment
- [ ] Code deployed to inactive
- [ ] Dependencies installed
- [ ] Module activated
- [ ] Smoke tests pass

### Switch
- [ ] Traffic switched
- [ ] Health checks pass
- [ ] Error rates normal
- [ ] Old environment kept warm

### Rollback
- [ ] Rollback script ready
- [ ] Previous version accessible
- [ ] Quick switch possible
