# WHMCS Load Testing Workflow

## Purpose

Comprehensive procedure for performing load and performance testing on WHMCS installations. Identifies bottlenecks, validates capacity planning, and ensures stability under realistic user loads.

## Prerequisites

- Load testing tools (Apache JMeter, k6, Locust, or similar)
- Monitoring tools (New Relic, Datadog, or Prometheus/Grafana)
- Test environment mirroring production
- Baseline performance metrics
- Test data representing realistic scenarios

## Workflow Steps

### Step 1: Define Performance Requirements

Establish clear performance goals:

```markdown
## Performance Requirements

**Target System:** WHMCS 8.x Production Clone
**Test Date:** YYYY-MM-DD

### Response Time SLAs:
- Homepage load: < 2 seconds
- Client portal login: < 1 second
- Invoice viewing: < 1.5 seconds
- Order completion: < 5 seconds
- API responses (average): < 500ms

### Throughput Targets:
- Concurrent users: 500
- Requests per second: 1000
- Peak load duration: 30 minutes

### Stability Requirements:
- No errors under normal load
- Graceful degradation under peak load
- Recovery time < 5 minutes after stress test
```

### Step 2: Prepare Test Environment

Set up isolated testing environment:

```bash
#!/bin/bash
# setup_load_test_environment.sh

# Clone production to test server
ssh prod-server "mysqldump -u root -p whmcs_main | gzip" | gunzip | mysql -u root -p whmcs_loadtest

# Sync files
rsync -avz --exclude='configuration.php' \
    prod-server:/var/www/whmcs/ \
    /var/www/whmcs-loadtest/

# Create test configuration
cat > /var/www/whmcs-loadtest/configuration.php << 'EOF'
<?php
$db_host = "localhost";
$db_username = "whmcs_loadtest";
$db_password = "loadtest_password";
$db_name = "whmcs_loadtest";
$cc_encryption_hash = 'your_encryption_hash';
$systems_url = 'http://loadtest.example.com';
$domain = 'loadtest.example.com';
$debug = false;
$display_errors = false;
EOF

# Generate test data if needed
php /var/www/whmcs-loadtest/resources/optimize_db.php
```

Create test user accounts:

```php
<?php
// generate_test_users.php
require_once __DIR__ . '/init.php';

$testUserCount = 1000;
$batchSize = 100;

for ($i = 1; $i <= $testUserCount; $i++) {
    $userData = [
        'firstname' => "LoadTest$i",
        'lastname' => "User",
        'email' => "loadtest$i@example.com",
        'company' => "LoadTest Company $i",
        'address1' => "$i Test Street",
        'city' => "TestCity",
        'state' => "TS",
        'country' => 'US',
        'postcode' => "12345",
        'phonenumber' => "+1-555-$i",
        'password' => md5('LoadTest123!'),
        'datecreated' => date('Y-m-d H:i:s'),
        'groupid' => 0,
    ];
    
    Capsule::table('tblclients')->insert($userData);
    
    if ($i % $batchSize === 0) {
        echo "Created $i users...\n";
    }
}

echo "Created $testUserCount test users";
```

### Step 3: Create Load Test Scripts

Configure load testing tool:

```javascript
// k6 load test script: whmcs_load_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
    stages: [
        { duration: '2m', target: 100 },   // Ramp up to 100 users
        { duration: '5m', target: 100 },   // Stay at 100 users
        { duration: '2m', target: 500 },    // Ramp to 500 users
        { duration: '10m', target: 500 },   // Stay at 500 users (peak)
        { duration: '2m', target: 0 },     // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<2000'],  // 95% under 2s
        http_req_failed: ['rate<0.01'],     // Less than 1% errors
        errors: ['rate<0.05'],              // Less than 5% errors
    },
};

const BASE_URL = 'https://loadtest.example.com';

export default function () {
    const scenarios = [
        () => testHomepage(),
        () => testClientLogin(),
        () => testInvoiceList(),
        () => testInvoiceView(),
        () => testProductList(),
        () => testCartCheckout(),
    ];
    
    // Random scenario selection weighted by expected traffic
    const weights = [30, 25, 15, 15, 10, 5];
    const random = Math.random() * 100;
    let cumulative = 0;
    
    for (let i = 0; i < weights.length; i++) {
        cumulative += weights[i];
        if (random < cumulative) {
            scenarios[i]();
            break;
        }
    }
    
    sleep(Math.random() * 3 + 1); // Random delay 1-4 seconds
}

function testHomepage() {
    const res = http.get(BASE_URL);
    check(res, {
        'homepage status 200': (r) => r.status === 200,
        'homepage loads quickly': (r) => r.timings.duration < 2000,
    });
    errorRate.add(res.status !== 200);
}

function testClientLogin() {
    // Use pre-created test accounts
    const credentials = {
        username: `loadtest${Math.floor(Math.random() * 1000)}@example.com`,
        password: 'LoadTest123!',
    };
    
    const res = http.post(`${BASE_URL}/dologin.php`, credentials);
    check(res, {
        'login successful': (r) => r.status === 200 || r.status === 302,
    });
    errorRate.add(res.status === 500 || res.status === 503);
}

function testInvoiceList() {
    const res = http.get(`${BASE_URL}/clientarea.php?action=invoices`);
    check(res, {
        'invoice list accessible': (r) => r.status === 200,
    });
    errorRate.add(res.status !== 200);
}

function testInvoiceView() {
    const invoiceId = Math.floor(Math.random() * 1000) + 1;
    const res = http.get(`${BASE_URL}/invoice.php?id=${invoiceId}`);
    check(res, {
        'invoice view accessible': (r) => r.status === 200,
    });
    errorRate.add(res.status !== 200);
}

function testProductList() {
    const res = http.get(`${BASE_URL}/cart.php?a=view`);
    check(res, {
        'product list accessible': (r) => r.status === 200,
    });
    errorRate.add(res.status !== 200);
}

function testCartCheckout() {
    // Step 1: Add product to cart
    const cartRes = http.post(`${BASE_URL}/cart.php?a=add&id=1`);
    
    // Step 2: Accept terms
    const acceptRes = http.post(`${BASE_URL}/cart.php?a=checkout`, {
        acceptsTerms: 1,
    });
    
    check(cartRes, {
        'add to cart works': (r) => r.status === 200,
    });
    errorRate.add(cartRes.status === 500);
}
```

### Step 4: Set Up Monitoring

Configure performance monitoring:

```yaml
# prometheus.yml configuration
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'whmcs'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'apache'
    static_configs:
      - targets: ['localhost:9117']
  
  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']
```

Create custom WHMCS metrics endpoint:

```php
<?php
// /var/www/whmcs/performance_metrics.php
require_once __DIR__ . '/init.php';

header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');

// Collect metrics
$metrics = [
    'timestamp' => time(),
    'php' => [
        'memory_usage' => memory_get_usage(true),
        'memory_peak' => memory_get_peak_usage(true),
        'execution_time' => microtime(true) - $_SERVER['REQUEST_TIME_FLOAT'],
    ],
    'database' => [
        'queries_total' => Capsule::table('tblactivitylog')->count(),
    ],
    'whmcs' => [
        'version' => \App::getVersion(),
        'active_clients' => Capsule::table('tblclients')
            ->where('status', 'Active')->count(),
        'active_services' => Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')->count(),
        'pending_orders' => Capsule::table('tblorders')
            ->where('status', 'Pending')->count(),
    ],
    'system' => [
        'load_avg' => sys_getloadavg()[0] ?? 0,
        'cpu_count' => PHP_INT_MAX, // indicator
    ],
];

// Database query performance
$queryTimes = Capsule::connection()->getElapsedQueryTime();
$metrics['database']['query_time_ms'] = $queryTimes * 1000;

// Cache statistics
try {
    $metrics['cache'] = [
        'driver' => config('cache.default'),
        'connected' => true,
    ];
} catch (Exception $e) {
    $metrics['cache'] = ['driver' => 'none', 'error' => $e->getMessage()];
}

echo json_encode($metrics);
```

### Step 5: Execute Load Tests

Run tests in phases:

```bash
# Phase 1: Baseline test (light load)
echo "=== Phase 1: Baseline Test ==="
k6 run --out json=results/baseline.json whmcs_load_test.js

# Phase 2: Expected peak load
echo "=== Phase 2: Peak Load Test ==="
k6 run --out json=results/peak.json whmcs_load_test.js

# Phase 3: Stress test (beyond expected)
echo "=== Phase 3: Stress Test ==="
k6 run --out json=results/stress.json --duration 5m --vus 1000 whmcs_stress_test.js

# Phase 4: Spike test
echo "=== Phase 4: Spike Test ==="
k6 run --out json=results/spike.json whmcs_spike_test.js
```

Monitor during tests:

```bash
# Terminal 1: Monitor server resources
watch -n 5 'echo "=== Server Stats ===" && \
    uptime && \
    echo "" && \
    echo "=== Memory ===" && \
    free -h && \
    echo "" && \
    echo "=== Disk I/O ===" && \
    iostat -x 1 1 && \
    echo "" && \
    echo "=== MySQL Connections ===" && \
    mysql -u root -p -e "SHOW STATUS LIKE Threads_connected";'

# Terminal 2: Monitor MySQL queries
mysql -u root -p -e "SHOW PROCESSLIST;" | wc -l

# Terminal 3: Monitor response times
tail -f /var/log/nginx/access.log | awk '{print $NF}' | sort | uniq -c | sort -rn | head
```

### Step 6: Analyze Results

Review test results:

```bash
# Generate test report
cat > generate_report.js << 'EOF'
import { readFileSync } from 'fs';

const baseline = JSON.parse(readFileSync('results/baseline.json', 'utf8'));
const peak = JSON.parse(readFileSync('results/peak.json', 'utf8'));

function calculateStats(data, metric) {
    const values = data.metrics[metric].values;
    return {
        avg: values.avg.toFixed(2),
        p95: values['p(95)'].toFixed(2),
        p99: values['p(99)'].toFixed(2),
        max: values.max.toFixed(2),
    };
}

console.log('=== Performance Report ===');
console.log('\n--- HTTP Request Duration ---');
console.log('Baseline:', calculateStats(baseline, 'http_req_duration'));
console.log('Peak:', calculateStats(peak, 'http_req_duration'));

console.log('\n--- Error Rates ---');
console.log('Baseline errors:', baseline.metrics.errors.values.rate.toFixed(4));
console.log('Peak errors:', peak.metrics.errors.values.rate.toFixed(4));

console.log('\n--- Requests Summary ---');
console.log('Baseline - Total:', baseline.metrics.http_reqs.values.count);
console.log('Peak - Total:', peak.metrics.http_reqs.values.count);
EOF

node generate_report.js
```

### Step 7: Identify Bottlenecks

Common WHMCS bottlenecks and diagnostics:

```php
// Database query analysis
// Add to configuration.php for query logging:
define('DB_QUERY_LOGGING', true);

// Review slow queries
mysql -u root -p -e "
SELECT 
    query,
    COUNT(*) AS executions,
    SUM(execution_time) AS total_time,
    AVG(execution_time) AS avg_time
FROM mysql.general_log
WHERE command_type = 'Query'
    AND argument LIKE 'SELECT%'
GROUP BY query
ORDER BY avg_time DESC
LIMIT 20;"

// Check for missing indexes
EXPLAIN SELECT * FROM tblhosting WHERE userid = 123;

<?php
// Check hook execution times
// Add to includes/lib.php:
add_hook('AfterCronJob', 1, function($vars) {
    $startTime = microtime(true);
    // Your cron logic
    $executionTime = microtime(true) - $startTime;
    
    if ($executionTime > 1) {
        logActivity("Slow cron detected: " . $executionTime . "s");
    }
});
```

### Step 8: Optimize and Retest

Implement optimizations:

```php
// Example: Optimize slow query with caching
<?php
// modules/custom/optimized_products.php

function getPopularProducts($limit = 10) {
    $cacheKey = 'popular_products_' . $limit;
    
    // Try cache first
    $cached = Cache::get($cacheKey);
    if ($cached !== null) {
        return $cached;
    }
    
    // Expensive query
    $products = Capsule::table('tblproducts')
        ->join('tblhosting', 'tblproducts.id', '=', 'tblhosting.packageid')
        ->selectRaw('tblproducts.*, COUNT(tblhosting.id) as order_count')
        ->groupBy('tblproducts.id')
        ->orderBy('order_count', 'desc')
        ->limit($limit)
        ->get();
    
    // Cache for 1 hour
    Cache::put($cacheKey, $products, 60);
    
    return $products;
}
```

```bash
# Apply MySQL optimizations
mysql -u root -p -e "
-- Add missing indexes
ALTER TABLE tblhosting ADD INDEX idx_userid_status (userid, domainstatus);
ALTER TABLE tblorders ADD INDEX idx_status_date (status, date);

-- Optimize tables
OPTIMIZE TABLE tblactivitylog;
"

# Clear application cache
php /var/www/whmcs/crons/cron.php?a=clearCache
```

### Step 9: Final Validation

Run final acceptance test:

```bash
# Final performance validation
k6 run --summary-export=results/final_summary.json whmcs_load_test.js

# Verify all SLAs met
php verify_sla.php
```

```php
<?php
// verify_sla.php
require_once __DIR__ . '/init.php';

$results = json_decode(file_get_contents('results/final_summary.json'), true);
$metrics = $results['metrics'];

$slaChecks = [
    'http_req_duration p95 < 2000ms' => $metrics['http_req_duration']['values']['p(95)'] < 2000,
    'http_req_duration p99 < 3000ms' => $metrics['http_req_duration']['values']['p(99)'] < 3000,
    'error rate < 1%' => $metrics['http_req_failed']['values']['rate'] < 0.01,
    'requests per second > 50' => $metrics['http_reqs']['values']['rate'] > 50,
];

$allPassed = true;
foreach ($slaChecks as $check => $passed) {
    $status = $passed ? '[PASS]' : '[FAIL]';
    echo "$status $check\n";
    if (!$passed) $allPassed = false;
}

if ($allPassed) {
    echo "\nAll SLA checks passed!\n";
    exit(0);
} else {
    echo "\nSome SLA checks failed.\n";
    exit(1);
}
```

## Verification Checklist

- [ ] Test environment matches production
- [ ] Test data is representative of production
- [ ] All test scenarios created
- [ ] Monitoring tools configured and running
- [ ] Baseline test completed
- [ ] Peak load test completed
- [ ] Stress test completed
- [ ] Results analyzed and documented
- [ ] Bottlenecks identified
- [ ] Optimizations implemented
- [ ] Retests completed after optimizations
- [ ] All SLAs met
- [ ] Performance report generated
- [ ] Recommendations documented

## Related Skills and Documentation

- [WHMCS Performance Audit](whmcs-performance-audit.md)
- [WHMCS Module Testing](whmcs-module-testing.md)
- [WHMCS Deployment Best Practices](whmcs-deployment-best-practices.md)
- WHMCS Performance Tips: https://docs.whmcs.com/Performance
- k6 Documentation: https://k6.io/docs/

## Notes

- Always test on isolated environments, never production
- Warm up cache before measuring performance
- Test at multiple times of day for varied conditions
- Document all findings for future reference
- Consider testing during actual peak periods if possible
