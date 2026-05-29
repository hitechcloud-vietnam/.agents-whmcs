# WHMCS Premium DNS Service Documentation

## Overview

Premium DNS is an advanced DNS hosting service with enhanced features including global anycast network, DDoS protection, 100% uptime SLA, and advanced traffic management.

## Features

### Core Features

| Feature | Basic DNS | Premium DNS |
|---------|-----------|-------------|
| Global Points of Presence | 3-5 locations | 30+ locations |
| Propagation Time | 24-48 hours | 15-30 minutes |
| Uptime SLA | 99% | 100% |
| DDoS Protection | None | Up to 400Gbps |
| Anycast Network | No | Yes (global) |
| Advanced Routing | No | Yes |
| DNSSEC | Optional | Included |
| Query Logs | No | Yes |
| API Access | Basic | Full |

## Configuration

### Enable Premium DNS

Navigate to: **Configuration > Products/Services > Premium DNS**

```php
// Premium DNS Configuration
$premiumDnsConfig = [
    // Service Settings
    'enabled' => true,
    'default_package' => 'standard',
    'allow_upgrade' => true,
    'allow_downgrade' => true,
    'trial_enabled' => true,
    'trial_days' => 14,

    // Network Settings
    'anycast_enabled' => true,
    'pop_locations' => [
        'us-east', 'us-west', 'eu-west', 'eu-central',
        'apac-east', 'apac-south', 'sa-east', 'au-east'
    ],
    'propagation_profile' => 'fast',

    // Protection Settings
    'ddos_protection' => true,
    'ddos_threshold' => '100Gbps',
    'rate_limiting' => true,
    'query_filtering' => false,

    // SLA Settings
    'sla_tier' => 'premium',
    'sla_uptime' => 100,
    'sla_credit_policy' => 'automatic'
];
```

### Package Tiers

```php
// Premium DNS Packages
$packages = [
    'starter' => [
        'name' => 'Premium DNS Starter',
        'price' => 4.99,
        'period' => 'monthly',
        'limits' => [
            'zones' => 5,
            'records_per_zone' => 500,
            'queries_per_month' => 5000000,
            'pops' => 10,
            'advanced_routing' => false
        ],
        'features' => [
            'anycast' => true,
            'ddos_protection' => '10Gbps',
            'dnssec' => true,
            'api' => 'basic'
        ]
    ],
    'professional' => [
        'name' => 'Premium DNS Professional',
        'price' => 14.99,
        'period' => 'monthly',
        'limits' => [
            'zones' => 25,
            'records_per_zone' => 2000,
            'queries_per_month' => 50000000,
            'pops' => 25,
            'advanced_routing' => true
        ],
        'features' => [
            'anycast' => true,
            'ddos_protection' => '100Gbps',
            'dnssec' => true,
            'geo_routing' => true,
            'latency_routing' => true,
            'api' => 'full'
        ]
    ],
    'enterprise' => [
        'name' => 'Premium DNS Enterprise',
        'price' => 49.99,
        'period' => 'monthly',
        'limits' => [
            'zones' => -1,  // unlimited
            'records_per_zone' => 10000,
            'queries_per_month' => -1,
            'pops' => 'all',
            'advanced_routing' => true
        ],
        'features' => [
            'anycast' => true,
            'ddos_protection' => '400Gbps',
            'dnssec' => true,
            'geo_routing' => true,
            'latency_routing' => true,
            'failover' => true,
            'traffic_policies' => true,
            'api' => 'full',
            'sla' => '100% uptime',
            'support' => '24/7 priority'
        ]
    ]
];
```

## Anycast Network

### How Anycast Works

With Anycast DNS, multiple servers share the same IP address. DNS queries are automatically routed to the nearest server based on network topology.

```
User (Tokyo) -> Query for example.com
       |
       v
   [Anycast IP 203.0.113.1]
       |
       v
[Routing discovers nearest PoP]
       |
       v
   PoP Tokyo (closest)
       |
       v
  Response served from Tokyo
```

### PoP Locations

```php
// Available Points of Presence
$popLocations = [
    // North America
    ['name' => 'New York', 'code' => 'nyc', 'region' => 'us-east'],
    ['name' => 'Los Angeles', 'code' => 'lax', 'region' => 'us-west'],
    ['name' => 'Seattle', 'code' => 'sea', 'region' => 'us-west'],
    ['name' => 'Chicago', 'code' => 'chi', 'region' => 'us-central'],
    ['name' => 'Miami', 'code' => 'mia', 'region' => 'us-south'],
    ['name' => 'Toronto', 'code' => 'tor', 'region' => 'ca-central'],

    // Europe
    ['name' => 'London', 'code' => 'lon', 'region' => 'eu-west'],
    ['name' => 'Amsterdam', 'code' => 'ams', 'region' => 'eu-west'],
    ['name' => 'Frankfurt', 'code' => 'fra', 'region' => 'eu-central'],
    ['name' => 'Paris', 'code' => 'par', 'region' => 'eu-west'],

    // Asia Pacific
    ['name' => 'Tokyo', 'code' => 'tyo', 'region' => 'apac-east'],
    ['name' => 'Singapore', 'code' => 'sin', 'region' => 'apac-south'],
    ['name' => 'Sydney', 'code' => 'syd', 'region' => 'apac-east'],
    ['name' => 'Mumbai', 'code' => 'bom', 'region' => 'apac-south'],
    ['name' => 'Seoul', 'code' => 'sel', 'region' => 'apac-east'],

    // South America
    ['name' => 'Sao Paulo', 'code' => 'gru', 'region' => 'sa-east']
];
```

### Network Status

```http
GET /premium-dns/network/status
```

**Response:**

```json
{
  "anycast_enabled": true,
  "active_pops": 28,
  "total_pops": 30,
  "pop_status": [
    {"code": "nyc", "status": "healthy", "load": 45, "queries_per_second": 12500},
    {"code": "lax", "status": "healthy", "load": 38, "queries_per_second": 9800},
    {"code": "tyo", "status": "healthy", "load": 52, "queries_per_second": 15200}
  ]
}
```

## DDoS Protection

### Protection Tiers

| Tier | Protection | Response Time | Cost Included |
|------|------------|---------------|---------------|
| Basic | 10 Gbps | 30 seconds | Starter+ |
| Professional | 100 Gbps | 10 seconds | Professional+ |
| Enterprise | 400 Gbps | Instant | Enterprise |

### Protection Settings

```php
// DDoS Protection Configuration
$ddosConfig = [
    'enabled' => true,
    'auto_mitigation' => true,
    'threshold_mbps' => 1000,
    'threshold_pps' => 100000,
    'blacklist_automation' => true,
    'notifications' => [
        'email' => true,
        'webhook' => true,
        'slack' => false
    ],
    'whitelist_ips' => [
        '192.0.2.0/24',
        '198.51.100.0/24'
    ]
];
```

### Attack Mitigation Log

```http
GET /premium-dns/ddos/logs
```

**Response:**

```json
{
  "attacks_blocked": 12,
  "attacks_this_month": 3,
  "largest_attack": {
    "date": "2024-01-15",
    "volume_gbps": 87,
    "duration_seconds": 120,
    "status": "mitigated"
  },
  "recent_mitigations": [
    {
      "id": "DMG-12345",
      "started_at": "2024-01-15T10:30:00Z",
      "ended_at": "2024-01-15T10:32:00Z",
      "volume_gbps": 87,
      "queries_blocked": 1500000
    }
  ]
}
```

## Advanced Routing

### Traffic Policies

```php
// Traffic Policy Configuration
$policy = WHMCS\Dns\TrafficPolicy::create([
    'name' => 'Multi-Region Distribution',
    'zone' => 'example.com',
    'rules' => [
        [
            'type' => 'geo',
            'match' => [
                ['region' => 'us', 'value' => '192.0.2.10']
            ],
            'fallback' => '192.0.2.1'
        ],
        [
            'type' => 'geo',
            'match' => [
                ['region' => 'eu', 'value' => '192.0.2.20']
            ],
            'fallback' => '192.0.2.1'
        ],
        [
            'type' => 'latency',
            'targets' => [
                ['ip' => '192.0.2.10', 'region' => 'us'],
                ['ip' => '192.0.2.20', 'region' => 'eu']
            ],
            'fallback' => '192.0.2.1'
        ]
    ]
]);
```

### Load Balancing

```php
// DNS Load Balancer
$lbPool = WHMCS\Dns\LoadBalancer::create([
    'name' => 'Web Server Pool',
    'zone' => 'example.com',
    'algorithm' => 'weighted_round_robin',
    'health_check' => [
        'enabled' => true,
        'type' => 'http',
        'endpoint' => '/health',
        'interval' => 30,
        'timeout' => 5,
        'retries' => 3
    ],
    'targets' => [
        [
            'ip' => '192.0.2.10',
            'weight' => 100,
            'region' => 'us-east',
            'enabled' => true
        ],
        [
            'ip' => '192.0.2.11',
            'weight' => 100,
            'region' => 'us-west',
            'enabled' => true
        ]
    ]
]);
```

### Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| Round Robin | Simple rotation | Even distribution |
| Weighted Round Robin | Weight-based rotation | Capacity-based distribution |
| Least Connections | Fewest active connections | Connection-oriented apps |
| Latency | Lowest latency target | Performance optimization |
| Geolocation | Geographic routing | Regional content |

## DNSSEC

### Enable DNSSEC

```php
// Enable DNSSEC for zone
$dnssec = WHMCS\Dns\DNSSEC::enable('example.com');

// Response
[
    'enabled' => true,
    'keys' => [
        [
            'id' => 'KSK-1',
            'type' => 'KSK',
            'algorithm' => 13,
            'key_tag' => 12345,
            'created_at' => '2024-01-15T00:00:00Z',
            'status' => 'active'
        ],
        [
            'id' => 'ZSK-1',
            'type' => 'ZSK',
            'algorithm' => 13,
            'key_tag' => 67890,
            'created_at' => '2024-01-15T00:00:00Z',
            'status' => 'active'
        ]
    ],
    'ds_records' => [
        'example.com. 3600 IN DS 12345 13 2 AABBCC...',
        'example.com. 3600 IN DS 67890 13 2 DDEEFF...'
    ]
];
```

### DS Record Setup

Provide DS records to your domain registrar:

```
DS Record 1:
Key Tag: 12345
Algorithm: 13 (ECDSAP256SHA256)
Digest Type: 2 (SHA-256)
Digest: AABBCCDDEEFF001122...

DS Record 2:
Key Tag: 67890
Algorithm: 13 (ECDSAP256SHA256)
Digest Type: 2 (SHA-256)
Digest: DDEEFFAABBCC001122...
```

## Analytics

### Query Analytics

```http
GET /premium-dns/analytics/queries
```

**Query Parameters:**
- `zone`: Filter by zone
- `period`: 1h, 24h, 7d, 30d, 90d
- `group_by`: minute, hour, day

**Response:**

```json
{
  "period": "24h",
  "total_queries": 2456789,
  "queries_by_type": {
    "A": 1456789,
    "AAAA": 523456,
    "MX": 123456,
    "TXT": 201234,
    "CNAME": 156789
  },
  "queries_by_pop": [
    {"pop": "nyc", "queries": 456789},
    {"pop": "tyo", "queries": 345678}
  ],
  "cache_hit_rate": 0.73,
  "response_time_ms": {
    "avg": 12,
    "p50": 8,
    "p95": 25,
    "p99": 45
  }
}
```

### Traffic Reports

```http
GET /premium-dns/reports/traffic
```

```json
{
  "top_domains": [
    {"domain": "example.com", "queries": 1234567},
    {"domain": "api.example.com", "queries": 456789}
  ],
  "top_record_types": [
    {"type": "A", "percentage": 59},
    {"type": "AAAA", "percentage": 21},
    {"type": "MX", "percentage": 5}
  ],
  "geographic_distribution": [
    {"region": "North America", "percentage": 45},
    {"region": "Europe", "percentage": 35},
    {"region": "Asia Pacific", "percentage": 18},
    {"region": "South America", "percentage": 2}
  ]
}
```

## SLA Credits

### Uptime Calculation

```php
// Calculate SLA compliance
$sla = WHMCS\PremiumDns\SLA::calculate([
    'zone' => 'example.com',
    'period' => 'monthly'
]);

// Response
[
    'period' => '2024-01',
    'uptime_percentage' => 100.0,
    'outage_minutes' => 0,
    'total_minutes' => 44640,
    'incidents' => [],
    'eligible_for_credit' => false,
    'credit_amount' => 0
];
```

### Credit Policy

| Uptime Tier | Credit Percentage |
|-------------|------------------|
| 100% | 0% (compliant) |
| 99.99% - 99.9% | 5% |
| 99.9% - 99% | 10% |
| 99% - 95% | 25% |
| Below 95% | 50% |

## API Reference

### Quick Reference

```http
GET    /premium-dns/zones                 # List zones
POST   /premium-dns/zones                 # Create zone
GET    /premium-dns/zones/{zone}          # Get zone
DELETE /premium-dns/zones/{zone}          # Delete zone

GET    /premium-dns/zones/{zone}/records  # List records
POST   /premium-dns/zones/{zone}/records  # Add record
PUT    /premium-dns/zones/{zone}/records/{id}  # Update record
DELETE /premium-dns/zones/{zone}/records/{id}  # Delete record

GET    /premium-dns/analytics/queries     # Query analytics
GET    /premium-dns/analytics/traffic     # Traffic report
GET    /premium-dns/ddos/status           # DDoS status
GET    /premium-dns/network/pops          # PoP locations
```

## See Also

- [Geo DNS Service](./whmcs-geo-dns-service.md)
- [Latency Routing](./whmcs-latency-routing.md)
- [Failover DNS](./whmcs-failover-dns.md)
