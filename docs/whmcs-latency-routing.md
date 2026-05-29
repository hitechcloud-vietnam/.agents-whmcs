# WHMCS Latency Routing Documentation

## Overview

Latency-based routing directs DNS queries to the endpoint with the lowest measured latency, ensuring optimal performance for users worldwide.

## How Latency Routing Works

### Resolution Flow

```
DNS Query Received
        |
        v
   Measure Latency to Each Endpoint
   (Real-time or cached measurements)
        |
        v
   Compare Latency Values
        |
        v
   Select Lowest Latency Endpoint
        |
        v
   Return Endpoint IP
        |
        v
   Cache Result for TTL
```

### Measurement Methods

| Method | Description | Accuracy |
|--------|-------------|----------|
| Active Probing | Periodic health checks | High |
| Passive | Based on historical data | Medium |
| Hybrid | Combined approach | Highest |

## Configuration

### Enable Latency Routing

Navigate to: **Configuration > Products/Services > Latency Routing**

```php
// Latency Routing Configuration
$latencyConfig = [
    'enabled' => true,
    'measurement_method' => 'active',
    'probe_interval' => 30,              // seconds
    'cache_ttl' => 60,                   // seconds
    'min_sample_size' => 5,              // measurements before using
    'timeout_ms' => 5000,                // probe timeout
    'fallback_to_geo' => true,
    'enable_ipv6' => true
];
```

### Endpoint Configuration

```php
// Latency Routing Endpoints
$endpoints = [
    'us-east' => [
        'host' => 'us-east.example.com',
        'ip' => '192.0.2.10',
        'ipv6' => '2001:db8::10',
        'region' => 'us-east',
        'weight' => 1.0,
        'enabled' => true
    ],
    'us-west' => [
        'host' => 'us-west.example.com',
        'ip' => '192.0.2.20',
        'ipv6' => '2001:db8::20',
        'region' => 'us-west',
        'weight' => 1.0,
        'enabled' => true
    ],
    'eu-west' => [
        'host' => 'eu-west.example.com',
        'ip' => '192.0.2.30',
        'ipv6' => '2001:db8::30',
        'region' => 'eu-west',
        'weight' => 1.0,
        'enabled' => true
    ],
    'apac-east' => [
        'host' => 'apac-east.example.com',
        'ip' => '192.0.2.40',
        'ipv6' => '2001:db8::40',
        'region' => 'apac-east',
        'weight' => 1.0,
        'enabled' => true
    ]
];
```

## Health Checks

### Health Check Configuration

```php
// Health Check Settings
$healthCheck = [
    'enabled' => true,
    'type' => 'http',              // http, https, tcp, icmp
    'endpoint' => '/health',
    'expected_status' => 200,
    'expected_response' => 'OK',
    'interval' => 30,              // seconds
    'timeout' => 5,                 // seconds
    'retries' => 3,
    'unhealthy_threshold' => 3,
    'healthy_threshold' => 2
];
```

### Health Check Types

#### HTTP/HTTPS Check

```php
$healthCheck = [
    'type' => 'https',
    'endpoint' => '/api/health',
    'method' => 'GET',
    'expected_status' => 200,
    'expected_body' => '{"status":"ok"}',
    'headers' => [
        'Host' => 'api.example.com'
    ]
];
```

#### TCP Check

```php
$healthCheck = [
    'type' => 'tcp',
    'port' => 443,
    'timeout' => 5
];
```

#### ICMP (Ping) Check

```php
$healthCheck = [
    'type' => 'icmp',
    'timeout' => 3,
    'count' => 3
];
```

### Health Status States

```
[Healthy] --(3 failures)--> [Unhealthy]
    ^                              |
    |                              |
    +--(2 successes)---------------+
```

## Latency-Based Records

### Create Latency Record

```http
POST /dns/zones/{zone}/records
```

**Request Body:**

```json
{
  "name": "api",
  "type": "A",
  "ttl": 60,
  "routing_type": "latency",
  "targets": [
    {
      "id": "target-1",
      "value": "192.0.2.10",
      "region": "us-east",
      "health_check": {
        "enabled": true,
        "type": "https",
        "endpoint": "/health"
      }
    },
    {
      "id": "target-2",
      "value": "192.0.2.20",
      "region": "us-west",
      "health_check": {
        "enabled": true,
        "type": "https",
        "endpoint": "/health"
      }
    },
    {
      "id": "target-3",
      "value": "192.0.2.30",
      "region": "eu-west",
      "health_check": {
        "enabled": true,
        "type": "https",
        "endpoint": "/health"
      }
    }
  ],
  "fallback": {
    "value": "192.0.2.1",
    "condition": "all_unhealthy"
  }
}
```

### Latency Response Example

```json
{
  "name": "api.example.com",
  "type": "A",
  "query_ip": "203.0.113.50",
  "measured_latency": {
    "us-east": 45,
    "us-west": 120,
    "eu-west": 180
  },
  "selected_target": {
    "value": "192.0.2.10",
    "region": "us-east",
    "latency_ms": 45
  },
  "cache_ttl": 60
}
```

## Load Balancing Algorithms

### Available Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| `latency` | Lowest latency | Performance |
| `latency_weighted` | Latency + weight | Capacity balancing |
| `round_robin` | Simple rotation | Even distribution |
| `weighted` | Weight-based | Multi-region capacity |

### Latency Weighted Algorithm

Combines latency with endpoint weights:

```php
$algorithm = [
    'type' => 'latency_weighted',
    'formula' => 'adjusted_latency = latency / weight',
    'min_latency_diff_ms' => 10  // Only switch if 10ms better
];
```

**Example:**
```
Endpoint A: Latency 50ms, Weight 1.0 -> Adjusted: 50ms
Endpoint B: Latency 60ms, Weight 2.0 -> Adjusted: 30ms

B selected despite higher raw latency due to higher capacity weight
```

## Failover Behavior

### Automatic Failover

```php
$failoverConfig = [
    'enabled' => true,
    'failover_order' => ['primary', 'secondary', 'tertiary'],
    'health_check_required' => true,
    'failback_enabled' => true,
    'failback_delay' => 300          // seconds after recovery
];
```

### Failover Decision Matrix

| Primary | Secondary | Tertiary | Action |
|---------|-----------|----------|--------|
| Healthy | Any | Any | Return primary |
| Unhealthy | Healthy | Any | Return secondary |
| Unhealthy | Unhealthy | Healthy | Return tertiary |
| All Unhealthy | All Unhealthy | All Unhealthy | Return fallback |

### Failover Flow

```
1. Health check fails for primary
2. Wait for unhealthy_threshold (3 failures)
3. Mark primary as unhealthy
4. Route to secondary (lowest latency among healthy)
5. Continue checking primary every 30s
6. When primary recovers, wait failback_delay
7. Switch back to primary
```

## Response Format

### Multi-Value Response

```json
{
  "name": "api.example.com",
  "type": "A",
  "ttl": 60,
  "values": [
    {
      "value": "192.0.2.10",
      "region": "us-east",
      "latency_ms": 45,
      "weight": 1.0,
      "health": "healthy"
    },
    {
      "value": "192.0.2.20",
      "region": "us-west",
      "latency_ms": 120,
      "weight": 1.0,
      "health": "healthy"
    }
  ],
  "returned_count": 2
}
```

### Single Value Response

```json
{
  "name": "api.example.com",
  "type": "A",
  "ttl": 60,
  "value": "192.0.2.10",
  "region": "us-east",
  "latency_ms": 45,
  "health": "healthy"
}
```

## Monitoring

### Latency Metrics

```http
GET /dns/latency-metrics?zone=example.com&period=24h
```

**Response:**

```json
{
  "zone": "example.com",
  "period": "24h",
  "endpoints": [
    {
      "endpoint": "192.0.2.10",
      "region": "us-east",
      "latency_ms": {
        "avg": 42,
        "min": 28,
        "max": 156,
        "p50": 38,
        "p95": 65,
        "p99": 120
      },
      "availability": 99.95,
      "queries_served": 2456789
    },
    {
      "endpoint": "192.0.2.20",
      "region": "us-west",
      "latency_ms": {
        "avg": 38,
        "min": 25,
        "max": 145,
        "p50": 35,
        "p95": 58,
        "p99": 110
      },
      "availability": 99.98,
      "queries_served": 1876543
    }
  ]
}
```

### Health Check History

```http
GET /dns/health-history?zone=example.com&endpoint=192.0.2.10
```

```json
{
  "endpoint": "192.0.2.10",
  "checks": [
    {
      "timestamp": "2024-01-15T10:30:00Z",
      "status": "healthy",
      "latency_ms": 42,
      "response_code": 200
    },
    {
      "timestamp": "2024-01-15T10:29:30Z",
      "status": "healthy",
      "latency_ms": 45,
      "response_code": 200
    }
  ],
  "failures": [
    {
      "timestamp": "2024-01-15T09:45:00Z",
      "status": "unhealthy",
      "error": "Connection timeout"
    }
  ]
}
```

## Web Interface

### Latency Routing Panel

**Configuration > Latency Routing > Manage**

```
+------------------------------------------------------------------+
| Latency Routing: api.example.com                                 |
+------------------------------------------------------------------+
| TTL: 60 | Algorithm: Latency | Failover: Enabled                  |
|                                                                  |
| Endpoints:                                                       |
| +--------------------------------------------------------------+ |
| | Region    | IP           | Latency | Health | Queries | Weight|| |
| |-----------|--------------|---------|--------|---------|-------|| |
| | US East   | 192.0.2.10   | 45ms    | OK     | 2.4M    | 1.0   || |
| | US West   | 192.0.2.20   | 38ms    | OK     | 1.8M    | 1.0   || |
| | EU West   | 192.0.2.30   | 180ms   | OK     | 0.9M    | 1.0   || |
| +--------------------------------------------------------------+ |
|                                                                  |
| Selected Endpoint: US West (38ms)                                |
|                                                                  |
| [Edit Endpoints] [Health Settings] [View Analytics]             |
+------------------------------------------------------------------+
```

### Real-Time Latency Map

```
+------------------------------------------------------------------+
|  Global Latency View                                    [Refresh]|
+------------------------------------------------------------------+
|                                                                  |
|  [Visual map showing latency from user locations to endpoints]   |
|                                                                  |
|  User Location: Tokyo, Japan                                     |
|                                                                  |
|  Endpoint Latency:                                               |
|  [US East   ] 180ms ████████████████████████████████████         |
|  [US West   ] 150ms ████████████████████████████████             |
|  [EU West   ] 220ms ██████████████████████████████████████████   |
|  [APAC East ]  25ms █████                                       |
|                                                                  |
|  Best Route: APAC East (if added as endpoint)                    |
+------------------------------------------------------------------+
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Inconsistent routing | Small sample size | Increase min_sample_size |
| Always same endpoint | Cache too long | Lower TTL |
| Slow health checks | Check interval too short | Balance speed vs load |
| Failover not working | Health check misconfigured | Verify check settings |
| High latency values | Network issues | Check endpoint status |

### Debug Commands

```bash
# Test latency routing
whmcscli dns latency-test --zone=example.com --name=api --ip=8.8.8.8

# Check endpoint health
whmcscli dns health-status --endpoint=192.0.2.10

# View latency measurements
whmcscli dns latency-stats --zone=example.com --period=1h

# Force health check
whmcscli dns health-check --endpoint=192.0.2.10 --force
```

## Best Practices

1. **Start with 3-4 endpoints** distributed globally
2. **Use appropriate TTL** - 60s for latency-sensitive, 300s+ for stable
3. **Configure meaningful health checks** that reflect true endpoint health
4. **Set up alerting** for health changes
5. **Test failover** periodically
6. **Monitor latency distribution** to optimize endpoint placement
7. **Use weighted latency** when endpoints have different capacities

## See Also

- [Geo DNS Service](./whmcs-geo-dns-service.md)
- [Failover DNS](./whmcs-failover-dns.md)
- [Premium DNS Service](./whmcs-premium-dns-service.md)
