# WHMCS Load Testing Workflow

## Purpose

Implement comprehensive load testing for WHMCS to validate system performance, identify bottlenecks, and ensure stability under production-level loads. This workflow covers test planning, execution, analysis, and optimization.

## Prerequisites

- Load testing tools (k6, Apache Bench, Locust)
- Monitoring infrastructure
- Test data preparation
- Performance baseline established

## Workflow Steps

### Step 1: Load Testing Planning

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHMCS Load Testing Plan                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │   Define Goals   │───▶│  Identify KPIs   │                   │
│  │  • Response time │    │  • RPS           │                   │
│  │  • Concurrent    │    │  • Error rate    │                   │
│  │  • Throughput    │    │  • Latency       │                   │
│  └──────────────────┘    └────────┬─────────┘                   │
│                                   │                             │
│                    ┌──────────────▼───────────────┐             │
│                    │     Design Test Scenarios    │             │
│                    │  • Normal usage patterns      │             │
│                    │  • Peak load scenarios        │             │
│                    │  • Stress test scenarios      │             │
│                    │  • Spike test scenarios       │             │
│                    └──────────────────┬───────────┘             │
│                                       │                         │
│                    ┌──────────────────▼───────────────┐         │
│                    │        Prepare Test Data         │         │
│                    │  • User accounts                 │         │
│                    │  • Product catalog              │         │
│                    │  • Historical data              │         │
│                    └──────────────────┬───────────┘             │
│                                       │                         │
│                    ┌──────────────────▼───────────────┐         │
│                    │        Execute Tests            │         │
│                    │  • Ramp-up phase                │         │
│                    │  • Steady-state phase           │         │
│                    │  • Cool-down phase              │         │
│                    └──────────────────┬───────────┘             │
│                                       │                         │
│                    ┌──────────────────▼───────────────┐         │
│                    │        Analyze Results          │         │
│                    │  • Performance metrics          │         │
│                    │  • Bottleneck identification     │         │
│                    │  • Optimization recommendations │         │
│                    └─────────────────────────────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Test Scenario Design

```javascript
// /opt/load-tests/whmcs-scenarios.js
// k6 load testing scenarios for WHMCS

import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const pageLoadTime = new Trend('page_load_time');
const apiResponseTime = new Trend('api_response_time');

// Configuration
const BASE_URL = 'https://whmcs.example.com';
const VUS = __ENV.VUS || 100;
const DURATION = __ENV.DURATION || '10m';

// Test scenarios
export let options = {
    stages: [
        { duration: '2m', target: VUS * 0.25 },   // Ramp up
        { duration: '5m', target: VUS },            // Steady state
        { duration: '2m', target: VUS * 1.5 },    // Stress
        { duration: '1m', target: 0 },             // Cool down
    ],
    thresholds: {
        http_req_duration: ['p(95)<2000', 'p(99)<5000'],
        http_req_failed: ['rate<0.01'],
        errors: ['rate<0.05'],
    },
};

// User simulation data
const userCredentials = [
    { email: 'user1@test.com', password: 'Test123!' },
    { email: 'user2@test.com', password: 'Test123!' },
    // ... more users
];

function login(email, password) {
    const res = http.post(`${BASE_URL}/dologin.php`, {
        email: email,
        password: password,
    }, {
        tags: { name: 'Login' },
    });
    
    check(res, {
        'login successful': (r) => r.url.includes('clientarea') || r.status === 200,
        'login response time': () => pageLoadTime.add(res.timings.duration),
    });
    
    return res;
}

export default function() {
    // Get random user
    const user = userCredentials[Math.floor(Math.random() * userCredentials.length)];
    
    group('Homepage', () => {
        const res = http.get(BASE_URL, { tags: { name: 'Homepage' } });
        check(res, {
            'homepage loaded': (r) => r.status === 200,
            'homepage fast': (r) => r.timings.duration < 2000,
        });
        pageLoadTime.add(res.timings.duration);
    });
    
    group('Client Area', () => {
        // Login
        const loginRes = login(user.email, user.password);
        
        if (loginRes.status === 200 || loginRes.url.includes('clientarea')) {
            // View dashboard
            const dashRes = http.get(`${BASE_URL}/clientarea.php`, {
                tags: { name: 'Dashboard' },
            });
            check(dashRes, { 'dashboard loaded': (r) => r.status === 200 });
            
            // View services
            const servicesRes = http.get(`${BASE_URL}/clientarea.php?action=services`, {
                tags: { name: 'Services' },
            });
            check(servicesRes, { 'services loaded': (r) => r.status === 200 });
            
            // View invoices
            const invoicesRes = http.get(`${BASE_URL}/clientarea.php?action=invoices`, {
                tags: { name: 'Invoices' },
            });
            check(invoicesRes, { 'invoices loaded': (r) => r.status === 200 });
        }
    });
    
    group('Product Catalog', () => {
        // Browse products
        const productsRes = http.get(`${BASE_URL}/cart.php`, {
            tags: { name: 'Cart' },
        });
        check(productsRes, {
            'cart page loaded': (r) => r.status === 200,
        });
        
        // Add to cart
        const addCartRes = http.post(`${BASE_URL}/cart.php?a=add&pid=1`, {
            tags: { name: 'AddToCart' },
        });
        check(addCartRes, {
            'added to cart': (r) => r.status === 200 || r.url.includes('checkout'),
        });
    });
    
    group('API Calls', () => {
        // Get client details via API
        const apiRes = http.get(`${BASE_URL}/includes/api.php`, {
            params: {
                action: 'GetClientsDetails',
                clientid: 1,
                username: 'admin',
                password: 'hash',
            },
            tags: { name: 'API' },
        });
        
        check(apiRes, {
            'api response': (r) => r.status === 200,
        });
        apiResponseTime.add(apiRes.timings.duration);
    });
    
    sleep(1);
}

// Stress test scenario
export function stressTest() {
    return {
        stages: [
            { duration: '5m', target: 1000 },   // Heavy load
            { duration: '10m', target: 1000 },   // Sustained
            { duration: '5m', target: 0 },      // Cool down
        ],
    };
}

// Spike test scenario
export function spikeTest() {
    return {
        stages: [
            { duration: '2m', target: 100 },     // Normal
            { duration: '30s', target: 2000 },   // Spike
            { duration: '5m', target: 100 },     // Recovery
        ],
    };
}
```

### Step 3: Database Load Testing

```sql
-- /opt/scripts/db-load-test.sql
-- WHMCS database performance testing queries

-- Simulate heavy query load
SELECT 
    COUNT(*) as total_invoices,
    SUM(total) as revenue
FROM tblinvoices
WHERE date >= DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Complex join query
SELECT 
    c.id as client_id,
    c.firstname,
    c.lastname,
    COUNT(DISTINCT h.id) as hosting_count,
    SUM(h.domain) as domain_count
FROM tblclients c
LEFT JOIN tblhosting h ON c.id = h.userid
LEFT JOIN tblproducts p ON h.packageid = p.id
WHERE c.status = 'Active'
GROUP BY c.id
HAVING COUNT(DISTINCT h.id) > 5
ORDER BY hosting_count DESC;

-- Inefficient query pattern (for testing)
SELECT * FROM tblorders o
LEFT JOIN tblorderitems oi ON o.id = oi.orderid
LEFT JOIN tblhosting h ON oi.relid = h.id
LEFT JOIN tblclients c ON h.userid = c.id
WHERE o.date >= '2025-01-01'
AND c.vat_id IS NOT NULL;

-- Bulk operations test
UPDATE tblhosting 
SET nextduedate = DATE_ADD(nextduedate, INTERVAL 1 MONTH)
WHERE nextduedate < NOW()
AND domain != '';
```

```bash
#!/bin/bash
# /opt/scripts/run-db-load-test.sh

MYSQL_HOST="localhost"
MYSQL_USER="whmcs_test"
MYSQL_PASS="test_password"
MYSQL_DB="whmcs_main"

echo "=== WHMCS Database Load Testing ==="
echo "Started: $(date)"
echo ""

# Concurrent query test
echo "Running 20 concurrent complex queries..."
for i in {1..20}; do
    mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASS $MYSQL_DB -e "
        SELECT COUNT(*) as count FROM tblclients WHERE status='Active';
        SELECT SUM(total) FROM tblinvoices WHERE status='Paid';
        SELECT COUNT(*) FROM tblhosting WHERE domain != '';
    " > /dev/null 2>&1 &
done

wait
echo "Concurrent queries complete."

# Connection limit test
echo ""
echo "Testing connection limits..."
for i in {1..100}; do
    mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASS -e "SELECT 1" > /dev/null 2>&1 &
done

wait
echo "Connection test complete."

# Monitor during test
echo ""
echo "=== Database Status During Test ==="
mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASS -e "
    SHOW STATUS LIKE 'Threads_connected';
    SHOW STATUS LIKE 'Threads_running';
    SHOW STATUS LIKE 'Innodb_row_lock_wait';
    SHOW FULL PROCESSLIST;
"
```

### Step 4: API Load Testing

```python
#!/usr/bin/env python3
# /opt/scripts/api-load-test.py

import concurrent.futures
import time
import requests
import statistics
from dataclasses import dataclass
from typing import List

@dataclass
class ApiTestResult:
    endpoint: str
    status_code: int
    response_time: float
    success: bool

class WHMCSLoadTester:
    def __init__(self, base_url: str, api_user: str, api_hash: str):
        self.base_url = base_url
        self.api_user = api_user
        self.api_hash = api_hash
        self.results: List[ApiTestResult] = []
    
    def make_api_call(self, action: str, **params) -> ApiTestResult:
        """Make a single API call and record results"""
        start = time.time()
        
        params.update({
            'username': self.api_user,
            'password': self.api_hash,
            'responsetype': 'json',
        })
        
        try:
            response = requests.post(
                f"{self.base_url}/includes/api.php",
                data=params,
                timeout=30
            )
            elapsed = time.time() - start
            
            return ApiTestResult(
                endpoint=action,
                status_code=response.status_code,
                response_time=elapsed,
                success=response.status_code == 200
            )
        except Exception as e:
            return ApiTestResult(
                endpoint=action,
                status_code=0,
                response_time=time.time() - start,
                success=False
            )
    
    def run_concurrent_test(self, action: str, concurrency: int, total_requests: int):
        """Run concurrent API test"""
        print(f"Running {total_requests} requests with {concurrency} concurrent workers...")
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=concurrency) as executor:
            futures = []
            
            for _ in range(total_requests):
                future = executor.submit(self.make_api_call, action)
                futures.append(future)
            
            for future in concurrent.futures.as_completed(futures):
                self.results.append(future.result())
        
        return self.analyze_results()
    
    def analyze_results(self) -> dict:
        """Analyze test results"""
        if not self.results:
            return {}
        
        response_times = [r.response_time for r in self.results]
        successes = [r for r in self.results if r.success]
        
        return {
            'total_requests': len(self.results),
            'successful': len(successes),
            'failed': len(self.results) - len(successes),
            'success_rate': len(successes) / len(self.results),
            'avg_response_time': statistics.mean(response_times),
            'median_response_time': statistics.median(response_times),
            'p95_response_time': sorted(response_times)[int(len(response_times) * 0.95)],
            'p99_response_time': sorted(response_times)[int(len(response_times) * 0.99)],
            'min_response_time': min(response_times),
            'max_response_time': max(response_times),
            'requests_per_second': len(self.results) / sum(response_times),
        }

# Test execution
if __name__ == '__main__':
    tester = WHMCSLoadTester(
        base_url='https://whmcs.example.com',
        api_user='admin',
        api_hash='your_api_hash'
    )
    
    # Test different API actions
    test_cases = [
        {'action': 'GetClientsDetails', 'concurrency': 50, 'total': 1000},
        {'action': 'GetInvoices', 'concurrency': 50, 'total': 1000},
        {'action': 'GetProducts', 'concurrency': 100, 'total': 2000},
    ]
    
    for test in test_cases:
        print(f"\nTesting {test['action']}...")
        results = tester.run_concurrent_test(
            test['action'],
            test['concurrency'],
            test['total']
        )
        
        print(f"Results: {results}")
        tester.results = []  # Reset for next test
```

### Step 5: Performance Monitoring During Tests

```yaml
# /opt/prometheus/whmcs-load-test-alerts.yml
# Monitoring rules for load testing

groups:
  - name: load_test_metrics
    interval: 5s
    rules:
      # Response time alerts
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m])) > 2
        for: 1m
        labels:
          severity: warning
          test: load
        annotations:
          summary: "High response time during load test"
          description: "P95 response time: {{ $value }}s"
      
      - alert: CriticalResponseTime
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m])) > 5
        for: 30s
        labels:
          severity: critical
          test: load
        annotations:
          summary: "Critical response time during load test"
      
      # Error rate alerts
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m]) > 0.01
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Error rate above 1% during load test"
      
      # Database performance
      - alert: DatabaseSlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High rate of slow queries during load test"
      
      # Connection pool alerts
      - alert: ConnectionPoolExhausted
        expr: phpfpm_activeProcesses / phpfpm_maxChildrenReached > 0.9
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "PHP-FPM connection pool nearly exhausted"
```

### Step 6: Load Test Analysis and Reporting

```python
#!/usr/bin/env python3
# /opt/scripts/analyze-load-results.py

import json
import sys
from datetime import datetime
from typing import Dict, List

class LoadTestAnalyzer:
    def __init__(self, results_file: str):
        self.results = self.load_results(results_file)
        self.baseline = self.load_baseline()
    
    def generate_report(self) -> str:
        """Generate comprehensive load test report"""
        
        report = []
        report.append(f"# WHMCS Load Test Report")
        report.append(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append("")
        
        # Summary
        report.append("## Summary")
        report.append(f"- Total Requests: {self.results['total_requests']:,}")
        report.append(f"- Successful: {self.results['successful']:,}")
        report.append(f"- Failed: {self.results['failed']:,}")
        report.append(f"- Success Rate: {self.results['success_rate']*100:.2f}%")
        report.append("")
        
        # Performance Metrics
        report.append("## Performance Metrics")
        report.append(f"- Average Response Time: {self.results['avg_response_time']*1000:.2f}ms")
        report.append(f"- Median Response Time: {self.results['median_response_time']*1000:.2f}ms")
        report.append(f"- P95 Response Time: {self.results['p95_response_time']*1000:.2f}ms")
        report.append(f"- P99 Response Time: {self.results['p99_response_time']*1000:.2f}ms")
        report.append(f"- Max Response Time: {self.results['max_response_time']*1000:.2f}ms")
        report.append(f"- Requests/Second: {self.results['requests_per_second']:.2f}")
        report.append("")
        
        # Baseline Comparison
        report.append("## Baseline Comparison")
        for metric, value in self.results.items():
            if metric in self.baseline:
                baseline = self.baseline[metric]
                diff = ((value - baseline) / baseline) * 100
                status = "OK" if abs(diff) < 20 else "DEGRADED"
                report.append(f"- {metric}: {value:.2f} (baseline: {baseline:.2f}, diff: {diff:+.1f}%) [{status}]")
        
        # Recommendations
        report.append("")
        report.append("## Recommendations")
        
        if self.results['p95_response_time'] > 2:
            report.append("- Consider optimizing slow queries or adding caching")
        
        if self.results['success_rate'] < 0.99:
            report.append("- Investigate and resolve failed requests")
        
        if self.results['requests_per_second'] < self.baseline.get('requests_per_second', 0):
            report.append("- Review configuration for throughput optimization")
        
        return "\n".join(report)
```

## Load Testing KPIs

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Response Time (P95) | <1s | >2s | >5s |
| Response Time (P99) | <2s | >5s | >10s |
| Error Rate | <0.1% | >1% | >5% |
| Requests/Second | >500 | >200 | >100 |
| CPU Usage | <70% | >85% | >95% |
| Memory Usage | <80% | >90% | >95% |
| DB Queries/Sec | <1000 | >1500 | >2000 |

## Best Practices

1. **Start Small**: Begin with lower load and incrementally increase
2. **Isolate Tests**: Run tests in controlled environment
3. **Measure Baseline**: Establish baseline before optimization
4. **Monitor All Components**: Web, database, cache, network
5. **Test Real Scenarios**: Simulate actual user behavior
6. **Document Results**: Track historical performance data
7. **Automate Tests**: Run tests regularly (nightly/weekly)

## Common Pitfalls

- **Incomplete Coverage**: Only testing homepage
- **Ignoring Database**: Not testing query performance
- **No Baseline**: No comparison point
- **Short Tests**: Not long enough to find memory leaks
- **Same Data**: Cache hits skew results
- **No Cleanup**: Test data affects subsequent tests

## Verification Checklist

- [ ] Test scenarios designed
- [ ] Load testing tools configured
- [ ] Test data prepared
- [ ] Baseline established
- [ ] Baseline load test executed
- [ ] Normal load test executed
- [ ] Peak load test executed
- [ ] Stress test executed
- [ ] Spike test executed
- [ ] Results analyzed
- [ ] Bottlenecks identified
- [ ] Optimization recommendations documented
- [ ] Capacity limits determined

## Related Documentation

- [WHMCS Scaling Guide](whmcs-scaling-guide.md)
- [WHMCS Capacity Planning](whmcs-capacity-planning.md)
- [WHMCS Performance Monitoring](whmcs-performance-monitoring.md)