# WHMCS TTL Management Documentation

## Overview

TTL (Time To Live) determines how long DNS records are cached by resolvers. Proper TTL management is critical for DNS propagation and service availability.

## Understanding TTL

### How TTL Works

```
1. First Query -> Resolver checks cache -> Cache MISS -> Query authoritative NS -> Cache result
2. Subsequent Queries -> Resolver checks cache -> Cache HIT -> Return cached result
3. TTL Expires -> Cache invalidated -> Next query triggers fresh lookup
```

### TTL Values

| Value | Duration | Use Case |
|-------|----------|----------|
| 60 | 1 minute | Rapid changes, testing |
| 300 | 5 minutes | Frequent updates |
| 900 | 15 minutes | Regular changes |
| 1800 | 30 minutes | Moderate stability |
| 3600 | 1 hour | Standard records |
| 7200 | 2 hours | Stable records |
| 14400 | 4 hours | Low change frequency |
| 86400 | 24 hours | Very stable |
| 604800 | 1 week | Minimal changes |

## Configuration

### Global TTL Settings

Navigate to: **Configuration > DNS Settings > TTL Configuration**

```php
// TTL Configuration
$ttlConfig = [
    // Default TTLs by record type
    'defaults' => [
        'A' => 3600,
        'AAAA' => 3600,
        'CNAME' => 3600,
        'MX' => 3600,
        'TXT' => 3600,
        'SRV' => 3600,
        'NS' => 86400,
        'SOA' => 3600,
        'CAA' => 3600
    ],

    // Minimum TTL
    'minimum' => 60,
    'maximum' => 604800,

    // TTL for new zones
    'new_zone_default' => 3600,

    // TTL reduction before changes
    'pre_change_reduction' => true,
    'pre_change_ttl' => 300,
    'post_change_ttl' => 3600,
    'pre_change_days' => 1
];
```

### TTL by Record Type

```php
// TTL presets
$ttlPresets = [
    'low' => 300,           // 5 minutes
    'standard' => 3600,     // 1 hour
    'high' => 14400,        // 4 hours
    'maximum' => 86400      // 24 hours
];

// Per-record-type TTL
$recordTtls = [
    'A' => [
        'default' => 3600,
        'minimum' => 60,
        'maximum' => 86400
    ],
    'MX' => [
        'default' => 3600,
        'minimum' => 300,
        'maximum' => 86400
    ],
    'TXT' => [
        'default' => 3600,
        'minimum' => 60,
        'maximum' => 86400
    ]
];
```

## TTL Strategies

### Standard Strategy

```php
// Standard TTL strategy
$standardStrategy = [
    'A' => 3600,          // Hourly refresh
    'AAAA' => 3600,
    'CNAME' => 3600,
    'MX' => 3600,
    'TXT' => 3600,
    'NS' => 86400
];
```

### High Availability Strategy

```php
// High availability TTL strategy
$haStrategy = [
    'A' => 60,            // Fast failover
    'AAAA' => 60,
    'CNAME' => 300,
    'MX' => 300,
    'TXT' => 300,
    'NS' => 3600
];
```

### CDN Strategy

```php
// CDN-optimized TTL strategy
$cdnStrategy = [
    'A' => 86400,         // Long cache (CDN handles changes)
    'AAAA' => 86400,
    'CNAME' => 86400,
    'TXT' => 3600         // SPF records more frequent
];
```

## Dynamic TTL Management

### Pre-Change TTL Reduction

Automatically reduce TTL before planned changes:

```php
// Auto TTL reduction configuration
$autoReduction = [
    'enabled' => true,
    'trigger_before_change' => true,
    'reduced_ttl' => 300,           // 5 minutes
    'reduction_duration' => 86400,  // 24 hours
    'restore_after_change' => true,
    'restore_ttl' => 3600
];
```

### TTL Schedule

```php
// Scheduled TTL changes
$ttlSchedule = [
    [
        'zone' => 'example.com',
        'record' => 'www',
        'current_ttl' => 86400,
        'scheduled_ttl' => 300,
        'scheduled_at' => '2024-01-20T00:00:00Z',
        'restore_at' => '2024-01-20T06:00:00Z',
        'restore_ttl' => 86400,
        'reason' => 'Planned migration'
    ]
];
```

## API Reference

### Update Record TTL

```http
PATCH /dns/zones/{zone}/records/{record_id}
```

**Request Body:**

```json
{
  "ttl": 1800
}
```

### Bulk TTL Update

```http
POST /dns/bulk/ttl
```

**Request Body:**

```json
{
  "domains": ["example.com", "example.net"],
  "filter": {
    "type": "A"
  },
  "ttl": 600,
  "reason": "Preparing for server migration"
}
```

### Schedule TTL Change

```http
POST /dns/ttl/schedule
```

**Request Body:**

```json
{
  "zone": "example.com",
  "record_id": "REC-001",
  "reduce_ttl": 300,
  "reduce_at": "2024-01-20T00:00:00Z",
  "restore_ttl": 3600,
  "restore_at": "2024-01-20T12:00:00Z"
}
```

## TTL and Propagation

### Propagation Time Reference

| TTL | Typical Full Propagation |
|-----|------------------------|
| 60 | 5-15 minutes |
| 300 | 15-30 minutes |
| 900 | 30-60 minutes |
| 1800 | 1-2 hours |
| 3600 | 2-4 hours |
| 7200 | 4-8 hours |
| 86400 | 8-24 hours |

### Pre-Migration Checklist

```php
// Migration TTL checklist
$migrationChecklist = [
    'days_before' => [
        3 => ['action' => 'Review current TTLs'],
        2 => ['action' => 'Schedule TTL reduction to 300s'],
        1 => ['action' => 'Verify TTL reduction applied']
    ],
    'migration_day' => [
        'hour_minus_1' => ['action' => 'Final verification'],
        'migration' => ['action' => 'Execute DNS changes'],
        'hour_plus_1' => ['action' => 'Verify changes propagated']
    ],
    'days_after' => [
        1 => ['action' => 'Restore TTLs to normal'],
        3 => ['action' => 'Verify TTL restoration']
    ]
];
```

## TTL Optimization

### Optimal TTL Calculator

```php
// TTL optimization based on change frequency
function calculateOptimalTTL($changeFrequency) {
    return match($changeFrequency) {
        'hourly' => 300,      // 5 minutes
        'daily' => 3600,     // 1 hour
        'weekly' => 14400,    // 4 hours
        'monthly' => 86400,   // 24 hours
        'rarely' => 172800,   // 48 hours
        default => 3600
    };
}
```

### TTL for Different Scenarios

| Scenario | Recommended TTL | Reasoning |
|----------|-----------------|-----------|
| Stable web server | 86400 | Rarely changes |
| Load balancer | 3600 | May change for maintenance |
| Failover target | 300 | Fast switchover needed |
| Blue-green deployment | 60 | Rapid traffic shifting |
| A/B testing | 300 | Frequent changes |
| API endpoint | 900 | Moderate stability |
| CDN origin | 86400 | Long cache at CDN |
| Mail server | 3600 | Stability important |

## Monitoring

### TTL Analytics

```http
GET /dns/analytics/ttl
```

**Response:**

```json
{
  "period": "30_days",
  "average_ttl_by_type": {
    "A": 3600,
    "AAAA": 3600,
    "CNAME": 3600,
    "MX": 3600,
    "TXT": 3600
  },
  "ttl_distribution": {
    "under_300": 5,
    "300_to_1800": 150,
    "1800_to_3600": 500,
    "over_3600": 45
  },
  "recommendations": [
    {
      "zone": "example.com",
      "record": "failover",
      "current_ttl": 86400,
      "recommended_ttl": 300,
      "reason": "Failover records should have lower TTL"
    }
  ]
}
```

## Customer Management

### Customer TTL Limits

```php
// Customer TTL restrictions
$customerLimits = [
    'allow_custom_ttl' => true,
    'min_ttl' => 300,
    'max_ttl' => 86400,
    'default_ttl' => 3600,
    'require_approval_over' => 43200  // 12 hours
];
```

### Customer TTL Selection

**Client Area > Domain Settings > TTL Configuration**

```
+------------------------------------------+
| TTL Configuration                        |
+------------------------------------------+
| Default TTL: [1 hour_____________]      |
|                                          |
| Individual Record TTL:                   |
| +--------------------------------------+|
| | Record    | Current | New TTL        ||
| |-----------|---------|----------------| |
| | @ (A)     | 3600    | [3600________] ||
| | www (CNAME)| 3600   | [3600________] ||
| | @ (MX)    | 3600    | [3600________] ||
| +--------------------------------------+|
|                                          |
| Quick Presets:                          |
| [Low (5min)] [Standard] [High (4hr)]     |
|                                          |
| [Save Changes]                          |
+------------------------------------------+
```

## Troubleshooting

### Common TTL Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Changes not propagating | High TTL | Reduce TTL before changes |
| Cache hit old IP | Resolver still cached | Wait for TTL expiry |
| Flapping DNS | TTL too low | Increase TTL |
| Failover slow | TTL too high | Reduce TTL for failover |

### TTL Debugging

```bash
# Check TTL for record
dig @ns1.registrar.com A example.com +noall +answer +ttlid

# Trace TTL propagation
whmcscli dns ttl-trace --zone=example.com --record=www

# Check resolver cache
dig @8.8.8.8 A example.com +noedns +noadflag
```

### TTL Verification

```php
// Verify TTL is set correctly
function verifyRecordTTL($zone, $recordId, $expectedTTL) {
    $record = getRecord($zone, $recordId);

    if ($record['ttl'] != $expectedTTL) {
        return [
            'success' => false,
            'error' => "TTL mismatch. Expected: {$expectedTTL}, Got: {$record['ttl']}"
        ];
    }

    return ['success' => true, 'ttl' => $record['ttl']];
}
```

## Best Practices

### TTL Guidelines

1. **Plan ahead** - Reduce TTL before planned changes
2. **Use appropriate values** - Match TTL to change frequency
3. **Monitor propagation** - Verify changes reach all resolvers
4. **Document TTL changes** - Track why TTL was modified
5. **Set minimum TTL** - Prevent extremely low values
6. **Use TTL presets** - Standardize TTL values

### TTL Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| TTL of 0 | Unpredictable caching | Use minimum 60 seconds |
| Very high TTL (week+) | Slow change propagation | Use reasonable values |
| Mixed TTL values | Confusion | Standardize per record type |
| No TTL monitoring | Unknown caching state | Implement TTL analytics |

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [Records Management](./whmcs-records-management.md)
- [Failover DNS](./whmcs-failover-dns.md)
