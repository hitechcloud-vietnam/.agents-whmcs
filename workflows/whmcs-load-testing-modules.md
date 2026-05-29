# WHMCS Load Testing Modules Workflow

## Overview
This workflow guides you through load testing WHMCS modules to ensure they perform well under high traffic and concurrent user scenarios.

## Prerequisites
- WHMCS installation (v8.0+)
- Apache Bench, k6, or Locust installed
- JMeter (optional)
- Monitoring tools (New Relic, Blackfire)
- Test server with production-like specs

## Step-by-Step Guide

### Step 1: Define Load Testing Goals

#### Performance Metrics to Measure
- Response time (average, p95, p99)
- Throughput (requests per second)
- Error rate
- Concurrent users
- Resource utilization (CPU, memory, database)

#### Baseline Metrics
```bash
# Establish baseline with current system
curl -w "@curl-format.txt" -o /dev/null -s http://your-whmcs-url/
```

### Step 2: Set Up k6 for Load Testing

#### Install k6
```bash
# macOS
brew install k6

# Linux
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# Windows
choco install k6
```

#### Create k6 Test Script
```javascript
// load-tests/whmcs-api-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const responseTime = new Trend('response_time');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    errors: ['rate<0.1'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://your-whmcs-url.com';

// Test scenarios
export default function () {
  const scenarios = [
    () => testHomePage(),
    () => testClientArea(),
    () => testCartPage(),
    () => testAPIEndpoint(),
    () => testAdminPanel(),
  ];

  const scenario = scenarios[Math.floor(Math.random() * scenarios.length)];
  scenario();
}

function testHomePage() {
  const res = http.get(`${BASE_URL}/`);
  const success = check(res, {
    'homepage status 200': (r) => r.status === 200,
    'homepage loads quickly': (r) => r.timings.duration < 500,
  });
  errorRate.add(!success);
  responseTime.add(res.timings.duration);
  sleep(Math.random() * 3 + 1);
}

function testClientArea() {
  const res = http.get(`${BASE_URL}/clientarea.php`);
  const success = check(res, {
    'clientarea status 200': (r) => r.status === 200,
    'login form present': (r) => r.body.includes('login'),
  });
  errorRate.add(!success);
  responseTime.add(res.timings.duration);
}

function testCartPage() {
  const res = http.get(`${BASE_URL}/cart.php`);
  const success = check(res, {
    'cart status 200': (r) => r.status === 200,
    'products displayed': (r) => r.body.includes('product'),
  });
  errorRate.add(!success);
  responseTime.add(res.timings.duration);
}

function testAPIEndpoint() {
  const payload = JSON.stringify({
    action: 'GetClients',
    username: 'api_user',
    password: 'api_password',
    accesskey: __ENV.API_KEY,
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
    },
  };

  const res = http.post(`${BASE_URL}/includes/api.php`, payload, params);
  const success = check(res, {
    'api status 200': (r) => r.status === 200,
    'api response valid': (r) => r.json('result') === 'success',
  });
  errorRate.add(!success);
  responseTime.add(res.timings.duration);
}

function testAdminPanel() {
  const res = http.get(`${BASE_URL}/admin/index.php`);
  const success = check(res, {
    'admin status 200': (r) => r.status === 200,
  });
  errorRate.add(!success);
  responseTime.add(res.timings.duration);
}
```

### Step 3: Test Module-Specific Endpoints
```javascript
// load-tests/module-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    acute_stress_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 50 },
        { duration: '1m', target: 50 },
        { duration: '30s', target: 0 },
      ],
    },
    soak_test: {
      executor: 'constant-vus',
      vus: 25,
      duration: '30m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.05'],
  },
};

const BASE_URL = __ENV.BASE_URL;

// Test module webhook endpoint
export function testWebhookEndpoint() {
  const payload = JSON.stringify({
    event: 'test_event',
    timestamp: Date.now(),
    data: { id: Math.floor(Math.random() * 1000) },
  });

  const signature = generateSignature(payload, __ENV.WEBHOOK_SECRET);

  const res = http.post(
    `${BASE_URL}/modules/addons/yourmodule/webhook.php`,
    payload,
    {
      headers: {
        'Content-Type': 'application/json',
        'X-Webhook-Signature': signature,
      },
    }
  );

  check(res, {
    'webhook responds': (r) => r.status >= 200 && r.status < 300,
    'webhook timing': (r) => r.timings.duration < 500,
  });
}

// Test module API endpoints
export function testModuleAPI() {
  const endpoints = [
    '/modules/addons/yourmodule/api/clients.php',
    '/modules/addons/yourmodule/api/sync.php',
    '/modules/addons/yourmodule/api/reports.php',
  ];

  const endpoint = endpoints[Math.floor(Math.random() * endpoints.length)];

  const res = http.get(`${BASE_URL}${endpoint}`, {
    headers: {
      'Authorization': `Bearer ${__ENV.API_TOKEN}`,
    },
  });

  check(res, {
    'api endpoint responds': (r) => r.status >= 200,
    'api response time': (r) => r.timings.duration < 1000,
  });
}

// Test heavy database queries
export function testDatabaseQuery() {
  const res = http.get(
    `${BASE_URL}/modules/addons/yourmodule/api/reports.php?action=export_all`
  );

  check(res, {
    'export completes': (r) => r.status === 200,
    'export time': (r) => r.timings.duration < 5000,
  });
}

function generateSignature(payload, secret) {
  // HMAC signature generation
  return 'sha256=' + require('crypto')
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
}
```

### Step 4: Run Load Tests

#### Basic Load Test
```bash
# Run with k6
k6 run load-tests/whmcs-api-load-test.js

# Run with environment variables
WHmcs_URL=http://localhost API_KEY=your_key k6 run load-tests/whmcs-api-load-test.js

# Run with cloud output
k6 run --out cloud load-tests/whmcs-api-load-test.js
```

#### Advanced Load Test
```bash
# Run specific test with custom config
k6 run \
  --vus 50 \
  --duration 60s \
  --iterations 1000 \
  --rate 100 \
  --batch 50 \
  load-tests/whmcs-api-load-test.js

# Run with InfluxDB output for Grafana
k6 run \
  --out influxdb=http://localhost:8086/k6 \
  load-tests/whmcs-api-load-test.js
```

### Step 5: Analyze Results

#### Key Metrics to Review
```
     http_req_duration..............: avg=245.31ms min=120.45ms
                                      med=230.12ms p(95)=480.23ms
                                      p(99)=750.45ms max=1200.12ms
     http_req_failed................: 0.5%
     vus............................: 100
     iterations.....................: 5000
```

#### Performance Benchmarks
| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Response Time (p95) | < 500ms | 500-1000ms | > 1000ms |
| Response Time (p99) | < 1000ms | 1000-2000ms | > 2000ms |
| Error Rate | < 1% | 1-5% | > 5% |
| Throughput | > 100 RPS | 50-100 RPS | < 50 RPS |

### Step 6: Apache Bench Alternative
```bash
# Basic benchmark
ab -n 1000 -c 100 http://your-whmcs-url.com/

# POST requests
ab -n 500 -c 50 -p data.txt -T application/json http://your-whmcs-url.com/api.php

# With cookies
ab -n 1000 -c 50 -C PHPSESSID=abc123 http://your-whmcs-url.com/clientarea.php
```

### Step 7: Continuous Load Testing
```yaml
# .github/workflows/load-tests.yml
name: Load Tests

on:
  schedule:
    - cron: '0 2 * * *'  # Run nightly at 2 AM
  workflow_dispatch:

jobs:
  load-test:
    runs-on: k6-runner
    steps:
      - uses: actions/checkout@v3

      - name: Run k6 load test
        uses: loadimpact/k6-load-test-with-reporter@v3
        with:
          filename: load-tests/whmcs-api-load-test.js
          cloud-token: ${{ secrets.K6_CLOUD_TOKEN }}
          env: |
            BASE_URL=${{ secrets.LOAD_TEST_URL }}
            API_KEY=${{ secrets.LOAD_TEST_API_KEY }}
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| High error rate | Check server logs, increase resources |
| Slow response times | Profile database queries, optimize caching |
| Connection timeouts | Increase connection pool, check network |
| Memory exhaustion | Monitor memory usage, fix leaks |
| Database bottlenecks | Optimize queries, add indexes |

## Load Test Report Template
```markdown
# Load Test Report - [Date]

## Environment
- Server: [Specs]
- WHMCS Version: [Version]
- Module Version: [Version]

## Test Configuration
- Duration: [X minutes]
- Max VUs: [Number]
- Test Type: [Soak/Stress/Spike]

## Results
- Total Requests: [Count]
- Failed Requests: [Count] ([Percentage])
- Average Response Time: [ms]
- p95 Response Time: [ms]
- p99 Response Time: [ms]
- Requests/sec: [RPS]

## Findings
### Pass
- [Positive finding]

### Failures
- [Issue description]

## Recommendations
1. [Recommendation]
```
