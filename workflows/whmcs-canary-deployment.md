# WHMCS Canary Deployment Workflow

## Overview
This workflow guides you through implementing canary deployment for WHMCS modules.

## Prerequisites
- Traffic splitting capability
- Monitoring tools
- Gradual rollout support

## Step-by-Step Guide

### Step 1: Set Up Canary Configuration
```php
// config/canary.php
<?php
return [
    'enabled' => true,
    'percentage' => 10,  // Start with 10%
    'gradual_increase' => [
        'step' => 10,       // Increase by 10%
        'interval' => 3600, // Every hour
    ],
    'target_percentage' => 100,
    'conditions' => [
        'min_error_rate' => 1,      // Max 1% errors
        'max_response_time' => 500, // Max 500ms
    ],
];
```

### Step 2: Create Canary Deployment Script
```bash
#!/bin/bash
# canary-deploy.sh

set -e

VERSION="${1}"
CURRENT_DIR="/var/www/whmcs/html/modules/addons/yourmodule"
CANARY_DIR="/var/www/whmcs-canary/html/modules/addons/yourmodule"

echo "=== Canary Deployment ==="
echo "Version: $VERSION"

# Deploy to canary environment
echo "[1/4] Deploying to canary..."
cp -r "$CURRENT_DIR" "$CANARY_DIR.bak"
rsync -av "$CURRENT_DIR/" "$CANARY_DIR/"

# Set canary percentage
echo "[2/4] Setting canary percentage to 10%..."
curl -X POST "http://localhost/admin/modules/addons/yourmodule/api.php" \
    -d "action=set-canary-percentage&percentage=10"

# Monitor
echo "[3/4] Monitoring for 1 hour..."
sleep 3600

# Check metrics
echo "[4/4] Checking metrics..."
curl -s "http://localhost/admin/modules/addons/yourmodule/api.php?action=canary-metrics"
```

### Step 3: Gradual Increase Script
```bash
#!/bin/bash
# canary-promote.sh

PERCENTAGE="${1:-10}"
MAX_PERCENTAGE=100
STEP=10

while [ "$PERCENTAGE" -le "$MAX_PERCENTAGE" ]; do
    echo "Increasing canary to ${PERCENTAGE}%..."
    
    # Update canary percentage
    curl -X POST "http://localhost/admin/modules/addons/yourmodule/api.php" \
        -d "action=set-canary-percentage&percentage=$PERCENTAGE"
    
    # Wait and check metrics
    sleep 3600
    
    # Get metrics
    METRICS=$(curl -s "http://localhost/admin/modules/addons/yourmodule/api.php?action=canary-metrics")
    ERROR_RATE=$(echo "$METRICS" | jq -r '.error_rate')
    AVG_RESPONSE=$(echo "$METRICS" | jq -r '.avg_response_time')
    
    echo "Metrics: Error Rate: $ERROR_RATE%, Response Time: ${AVG_RESPONSE}ms"
    
    # Check if metrics are acceptable
    if (( $(echo "$ERROR_RATE > 1" | bc -l) )) || [ "$AVG_RESPONSE" -gt 500 ]; then
        echo "ERROR: Metrics exceeded thresholds. Rolling back canary."
        curl -X POST "http://localhost/admin/modules/addons/yourmodule/api.php" \
            -d "action=set-canary-percentage&percentage=0"
        exit 1
    fi
    
    PERCENTAGE=$((PERCENTAGE + STEP))
done

echo "Canary deployment complete - 100% traffic on new version"
```

### Step 4: Monitor Canary
```php
// api/canary-metrics.php
<?php
// Get canary metrics

$startTime = time() - 3600; // Last hour
$endTime = time();

$metrics = [
    'requests' => 0,
    'errors' => 0,
    'total_response_time' => 0,
];

// Query logs for canary requests
$canaryLogs = \WHMCS\Database\Capsule::table('mod_yourmodule_logs')
    ->where('is_canary', true)
    ->whereBetween('created_at', [$startTime, $endTime])
    ->get();

foreach ($canaryLogs as $log) {
    $metrics['requests']++;
    $metrics['total_response_time'] += $log['response_time'];
    if ($log['status'] === 'error') {
        $metrics['errors']++;
    }
}

$metrics['error_rate'] = ($metrics['requests'] > 0) 
    ? ($metrics['errors'] / $metrics['requests']) * 100 
    : 0;
$metrics['avg_response_time'] = ($metrics['requests'] > 0) 
    ? $metrics['total_response_time'] / $metrics['requests'] 
    : 0;

echo json_encode($metrics);
```

### Step 5: Execute Canary Deployment
```bash
# Deploy new version to canary
./canary-deploy.sh v2.0.0

# Monitor and promote
./canary-promote.sh

# Or abort if issues
curl -X POST "http://localhost/admin/modules/addons/yourmodule/api.php" \
    -d "action=set-canary-percentage&percentage=0"
```

## Canary Deployment Checklist

### Setup
- [ ] Canary environment ready
- [ ] Traffic splitting configured
- [ ] Monitoring set up
- [ ] Rollback plan ready

### Initial Deploy
- [ ] Canary deployed
- [ ] Small percentage set
- [ ] Metrics monitored
- [ ] No errors

### Gradual Increase
- [ ] Metrics acceptable
- [ ] Percentage increased
- [ ] Continue monitoring
- [ ] Issues addressed

### Full Rollout
- [ ] 100% reached
- [ ] Canary disabled
- [ ] Cleanup completed
- [ ] Documentation updated
