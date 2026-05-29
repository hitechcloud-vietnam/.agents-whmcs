# WHMCS Failover DNS Documentation

## Overview

DNS Failover automatically monitors your endpoints and routes traffic to backup servers when primary servers become unavailable, ensuring high availability for your services.

## How DNS Failover Works

### Failover Flow

```
DNS Query
    |
    v
Primary Server
    |
    +--[Health Check OK]--> Return Primary IP
    |
    +--[Health Check Fail]--> Failover Triggered
                                |
                                v
                           Secondary Server
                                |
                                +--[Secondary OK]--> Return Secondary IP
                                |
                                +--[Secondary Fail]--> Continue to Tertiary
```

## Configuration

### Enable DNS Failover

Navigate to: **Configuration > Products/Services > DNS Failover**

```php
// DNS Failover Configuration
$failoverConfig = [
    'enabled' => true,
    'failover_timeout' => 10,            // seconds
    'health_check_interval' => 30,         // seconds
    'unhealthy_threshold' => 3,            // consecutive failures
    'healthy_threshold' => 2,              // consecutive successes
    'failback_enabled' => true,
    'failback_delay' => 300,               // seconds
    'notifications_enabled' => true
];
```

### Failover Pools

```php
// Define Failover Pool
$failoverPool = WHMCS\Dns\FailoverPool::create([
    'name' => 'Web Server Pool',
    'zone' => 'example.com',
    'record_name' => 'www',
    'record_type' => 'A',
    'ttl' => 60,
    'health_check' => [
        'type' => 'https',
        'endpoint' => '/health',
        'expected_status' => 200,
        'timeout' => 5,
        'interval' => 30
    ],
    'targets' => [
        [
            'id' => 'primary',
            'ip' => '192.0.2.10',
            'region' => 'us-east',
            'priority' => 1,
            'enabled' => true
        ],
        [
            'id' => 'secondary',
            'ip' => '192.0.2.20',
            'region' => 'us-west',
            'priority' => 2,
            'enabled' => true
        ],
        [
            'id' => 'tertiary',
            'ip' => '192.0.2.30',
            'region' => 'eu-west',
            'priority' => 3,
            'enabled' => true
        ]
    ]
]);
```

## Health Checks

### Health Check Types

| Type | Protocol | Use Case |
|------|----------|----------|
| HTTP | Port 80/443 | Web applications |
| HTTPS | Port 443 | Secure web apps |
| TCP | Custom port | Non-HTTP services |
| ICMP | Ping | Basic connectivity |
| DNS | Port 53 | DNS servers |
| SMTP | Port 25/587 | Email servers |

### HTTP Health Check

```php
$healthCheck = [
    'type' => 'http',
    'host' => 'www.example.com',
    'port' => 443,
    'path' => '/health',
    'method' => 'GET',
    'expected_status' => 200,
    'expected_body' => 'OK',
    'headers' => [
        'Host' => 'www.example.com',
        'User-Agent' => 'HealthCheck/1.0'
    ],
    'timeout' => 5,
    'interval' => 30,
    'retries' => 3
];
```

### HTTPS Health Check with SSL

```php
$healthCheck = [
    'type' => 'https',
    'host' => 'api.example.com',
    'port' => 443,
    'path' => '/health',
    'method' => 'GET',
    'expected_status' => 200,
    'verify_ssl' => true,
    'timeout' => 5,
    'interval' => 30
];
```

### TCP Health Check

```php
$healthCheck = [
    'type' => 'tcp',
    'host' => 'smtp.example.com',
    'port' => 587,
    'timeout' => 5,
    'interval' => 30
];
```

### ICMP (Ping) Health Check

```php
$healthCheck = [
    'type' => 'icmp',
    'host' => '192.0.2.10',
    'timeout' => 3,
    'interval' => 30,
    'count' => 3
];
```

### DNS Health Check

```php
$healthCheck = [
    'type' => 'dns',
    'host' => 'ns1.example.com',
    'query_name' => 'healthcheck.example.com',
    'query_type' => 'A',
    'expected_response' => '192.0.2.100',
    'timeout' => 5,
    'interval' => 60
];
```

## Failover Rules

### Create Failover Rule

```http
POST /dns/failover/rules
```

**Request Body:**

```json
{
  "name": "www",
  "type": "A",
  "zone": "example.com",
  "failover_enabled": true,
  "targets": [
    {
      "id": "primary",
      "ip": "192.0.2.10",
      "priority": 1,
      "health_check": {
        "type": "https",
        "endpoint": "/health",
        "expected_status": 200
      }
    },
    {
      "id": "backup",
      "ip": "192.0.2.20",
      "priority": 2,
      "health_check": {
        "type": "https",
        "endpoint": "/health",
        "expected_status": 200
      }
    }
  ],
  "fallback": {
    "ip": "192.0.2.1",
    "condition": "all_unhealthy"
  }
}
```

### Failover Behavior Matrix

| Primary | Secondary | Tertiary | Response |
|---------|-----------|----------|----------|
| Healthy | Any | Any | Primary |
| Unhealthy | Healthy | Any | Secondary |
| Unhealthy | Unhealthy | Healthy | Tertiary |
| Unhealthy | Unhealthy | Unhealthy | Fallback |
| No fallback configured | - | - | NXDOMAIN or SERVFAIL |

### State Transitions

```
[Healthy] --(3 failures)--> [Unhealthy]
     ^                           |
     |                           |
     |                           v
     +---(2 successes)----[Healthy]
```

## Notification Settings

### Configure Notifications

```php
$notifications = [
    'enabled' => true,
    'channels' => ['email', 'webhook', 'sms'],
    'events' => [
        'target_down' => true,
        'target_up' => true,
        'failover_triggered' => true,
        'failback_completed' => true
    ],
    'email' => [
        'recipients' => ['admin@example.com', 'ops@example.com'],
        'template' => 'failover_notification'
    ],
    'webhook' => [
        'url' => 'https://your-app.com/webhooks/failover',
        'method' => 'POST',
        'headers' => ['Authorization' => 'Bearer xxx']
    ],
    'sms' => [
        'recipients' => ['+1234567890'],
        'critical_only' => false
    ]
];
```

### Webhook Payload

```json
{
  "event": "failover_triggered",
  "timestamp": "2024-01-15T10:30:00Z",
  "zone": "example.com",
  "record": "www",
  "record_type": "A",
  "previous_target": {
    "id": "primary",
    "ip": "192.0.2.10",
    "reason": "health_check_failed"
  },
  "new_target": {
    "id": "secondary",
    "ip": "192.0.2.20"
  },
  "health_check_details": {
    "check_type": "https",
    "failures": 3,
    "last_error": "Connection timeout"
  }
}
```

## Monitoring

### Failover Status

```http
GET /dns/failover/status/{zone}
```

**Response:**

```json
{
  "zone": "example.com",
  "record": "www",
  "record_type": "A",
  "current_target": {
    "id": "secondary",
    "ip": "192.0.2.20",
    "priority": 2
  },
  "pool_status": [
    {
      "id": "primary",
      "ip": "192.0.2.10",
      "status": "unhealthy",
      "last_check": "2024-01-15T10:29:30Z",
      "failures": 5,
      "consecutive_failures": 3
    },
    {
      "id": "secondary",
      "ip": "192.0.2.20",
      "status": "healthy",
      "last_check": "2024-01-15T10:30:00Z",
      "failures": 0
    },
    {
      "id": "tertiary",
      "ip": "192.0.2.30",
      "status": "healthy",
      "last_check": "2024-01-15T10:30:00Z",
      "failures": 0
    }
  ],
  "failover_history": [
    {
      "timestamp": "2024-01-15T10:30:00Z",
      "from": "primary",
      "to": "secondary",
      "reason": "health_check_failed"
    }
  ]
}
```

### Health Check Logs

```http
GET /dns/failover/health-logs/{pool_id}
```

```json
{
  "pool_id": "POOL-12345",
  "logs": [
    {
      "timestamp": "2024-01-15T10:30:00Z",
      "target": "192.0.2.10",
      "status": "unhealthy",
      "response_time_ms": null,
      "error": "Connection refused"
    },
    {
      "timestamp": "2024-01-15T10:29:30Z",
      "target": "192.0.2.10",
      "status": "unhealthy",
      "response_time_ms": null,
      "error": "Connection timeout"
    },
    {
      "timestamp": "2024-01-15T10:29:00Z",
      "target": "192.0.2.10",
      "status": "unhealthy",
      "response_time_ms": null,
      "error": "Connection timeout"
    },
    {
      "timestamp": "2024-01-15T10:28:30Z",
      "target": "192.0.2.10",
      "status": "healthy",
      "response_time_ms": 45
    }
  ]
}
```

## DNS Record Response

### During Normal Operation

```json
{
  "question": {
    "name": "www.example.com",
    "type": "A"
  },
  "answer": {
    "name": "www.example.com",
    "type": "A",
    "ttl": 60,
    "value": "192.0.2.10",
    "source": "primary"
  }
}
```

### During Failover

```json
{
  "question": {
    "name": "www.example.com",
    "type": "A"
  },
  "answer": {
    "name": "www.example.com",
    "type": "A",
    "ttl": 60,
    "value": "192.0.2.20",
    "source": "secondary",
    "failover_active": true
  },
  "metadata": {
    "primary_status": "unhealthy",
    "failover_reason": "health_check_failed"
  }
}
```

## Web Interface

### Failover Dashboard

**Configuration > DNS Failover > Dashboard**

```
+------------------------------------------------------------------+
|  DNS Failover Status                              [Refresh: 30s]  |
+------------------------------------------------------------------+
|                                                                  |
|  Active Failovers: 1                                             |
|  Total Pools: 5                                                 |
|  Healthy Targets: 12/15                                          |
|                                                                  |
|  Pool: Web Servers (example.com - www)                          |
|  +--------------------------------------------------------------+|
|  | Target     | IP          | Priority | Status  | Last Check   ||
|  |------------|-------------|----------|---------|-------------||
|  | Primary    | 192.0.2.10  | 1        | UNHEALTHY| 10:30:00    ||
|  | Secondary  | 192.0.2.20  | 2        | HEALTHY  | 10:30:00    ||
|  | Tertiary  | 192.0.2.30  | 3        | HEALTHY  | 10:30:00    ||
|  +--------------------------------------------------------------+|
|  |                                                                  |
|  | Current Response: 192.0.2.20 (Secondary)                         |
|  | Failover Active: Yes                                            |
|  | Reason: Primary health check failed                              |
|  | Since: 10:30:00 (0 minutes ago)                                  |
|  |                                                                  |
|  | [Manually Switch] [Check Now] [Edit Pool]                        |
|  +------------------------------------------------------------------+
```

### Failover Event Timeline

```
+------------------------------------------------------------------+
|  Failover Events (last 24 hours)                                  |
+------------------------------------------------------------------+
|                                                                  |
|  10:30:00 - Failover: Primary (192.0.2.10) -> Secondary (192.0.2.20)
|            Reason: Health check failed (3 consecutive failures)   |
|                                                                  |
|  10:29:30 - Health Check Failed: 192.0.2.10 (Connection refused) |
|  10:29:00 - Health Check Failed: 192.0.2.10 (Timeout)           |
|  10:28:30 - Health Check Failed: 192.0.2.10 (Timeout)            |
|  10:28:00 - Health Check OK: 192.0.2.10 (Response: 45ms)        |
|                                                                  |
+------------------------------------------------------------------+
```

## Manual Operations

### Force Failover

```http
POST /dns/failover/{pool_id}/switch
```

**Request Body:**

```json
{
  "target_id": "secondary",
  "reason": "manual_intervention",
  "temporary": false
}
```

### Cancel Failover

```http
POST /dns/failover/{pool_id}/cancel
```

### Pause Health Checks

```http
POST /dns/failover/{pool_id}/pause
```

**Request Body:**

```json
{
  "duration": 3600,
  "reason": "Scheduled maintenance"
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Failover not triggering | Health check misconfigured | Verify check settings |
| Flapping (rapid on/off) | Unstable endpoint | Increase threshold |
| All targets unhealthy | Widespread outage | Check external status |
| Stuck in failover | Failback disabled | Enable failback |
| DNS still pointing to down | High TTL | Lower TTL before maintenance |

### Debug Commands

```bash
# Check failover status
whmcscli dns failover status --zone=example.com --record=www

# Force health check
whmcscli dns failover health-check --pool_id=POOL-12345 --target=primary --force

# Switch target manually
whmcscli dns failover switch --pool_id=POOL-12345 --target=secondary

# View failover logs
whmcscli dns failover logs --zone=example.com --period=24h
```

## Best Practices

1. **Always have 2+ backup targets** - Single backup is a single point of failure
2. **Use geographic diversity** - Spread targets across regions
3. **Configure meaningful health checks** - Check actual service availability
4. **Set appropriate thresholds** - Balance speed of detection vs false positives
5. **Test failover regularly** - Simulate failures to verify behavior
6. **Monitor health check latency** - Slow checks mean slow failover
7. **Document runbooks** - Know what to do when failover occurs

## See Also

- [Geo DNS Service](./whmcs-geo-dns-service.md)
- [Latency Routing](./whmcs-latency-routing.md)
- [Premium DNS Service](./whmcs-premium-dns-service.md)
