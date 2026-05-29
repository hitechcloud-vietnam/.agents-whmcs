# WHMCS Geo DNS Service Documentation

## Overview

Geo DNS (Geographic DNS) routes DNS queries to different servers based on the geographic location of the resolver. This enables localized content delivery and improved performance.

## How Geo DNS Works

### Resolution Flow

```
DNS Query from User
        |
        v
   GeoIP Detection
   (Determine user location)
        |
        v
   Policy Evaluation
   (Match location to rule)
        |
        v
   Target Selection
   (Return appropriate IP)
        |
        v
   DNS Response
   (Location-specific answer)
```

### GeoIP Databases

```php
// GeoIP Configuration
$geoipConfig = [
    'provider' => 'maxmind',
    'database' => 'GeoLite2-City',
    'update_frequency' => 'monthly',
    'fallback' => 'default_record'
];
```

## Configuration

### Enable Geo DNS

Navigate to: **Configuration > Products/Services > Geo DNS**

```php
// Geo DNS Configuration
$geoDnsConfig = [
    'enabled' => true,
    'default_action' => 'return_default',
    'fallback_enabled' => true,
    'unknown_location' => 'default',
    'cache_ttl' => 300,
    'ipv6_support' => true,
    'edns_client_subnet' => true
];
```

### Regional Mapping

```php
// Geographic Regions
$regions = [
    'us-east' => ['US', 'CA'],           // US East Coast
    'us-west' => ['US-CA'],              // US West Coast
    'us-central' => ['US-AZ', 'US-CO'],  // US Central
    'eu-west' => ['GB', 'IE', 'FR'],    // Western Europe
    'eu-central' => ['DE', 'AT', 'CH'], // Central Europe
    'eu-north' => ['SE', 'NO', 'FI'],   // Northern Europe
    'apac-east' => ['JP', 'KR', 'TW'], // East Asia
    'apac-south' => ['IN', 'SG', 'TH'], // South Asia
    'apac-se' => ['AU', 'NZ'],          // Oceania
    'sa-east' => ['BR', 'AR', 'CL'],    // South America
    'africa' => ['ZA', 'EG', 'NG']      // Africa
];
```

## Geo Routing Rules

### Create Geo Rule

```http
POST /dns/zones/{zone}/geo-rules
```

**Request Body:**

```json
{
  "name": "@",
  "type": "A",
  "rules": [
    {
      "region": "us-east",
      "value": "192.0.2.10",
      "priority": 10
    },
    {
      "region": "us-west",
      "value": "192.0.2.20",
      "priority": 10
    },
    {
      "region": "eu-west",
      "value": "192.0.2.30",
      "priority": 10
    },
    {
      "region": "apac-east",
      "value": "192.0.2.40",
      "priority": 10
    }
  ],
  "default": {
    "value": "192.0.2.1",
    "regions": ["_unknown", "_default"]
  }
}
```

### Rule Types

| Rule Type | Description | Example |
|-----------|-------------|---------|
| Country | Specific country code | `US`, `GB`, `JP` |
| Region | Geographic region | `eu-west`, `apac-east` |
| Continent | Continent level | `NA`, `EU`, `AS` |
| ASN | Autonomous System Number | `AS15169` (Google) |
| IP Range | CIDR notation | `192.0.2.0/24` |
| ISP | Internet provider | `comcast`, `at&t` |

### Country Codes

```php
// Common Country Mappings
$countryToRegion = [
    // North America
    'US' => 'us-east',
    'CA' => 'us-east',
    'MX' => 'us-south',

    // Europe
    'GB' => 'eu-west',
    'DE' => 'eu-central',
    'FR' => 'eu-west',
    'NL' => 'eu-west',
    'SE' => 'eu-north',
    'NO' => 'eu-north',

    // Asia Pacific
    'JP' => 'apac-east',
    'KR' => 'apac-east',
    'CN' => 'apac-east',
    'IN' => 'apac-south',
    'SG' => 'apac-south',
    'AU' => 'apac-se',

    // South America
    'BR' => 'sa-east',
    'AR' => 'sa-east',
    'CL' => 'sa-east'
];
```

## DNS Record Structure

### A Record with Geo Rules

```json
{
  "name": "www",
  "type": "A",
  "ttl": 300,
  "geo_rules": {
    "default": {"value": "192.0.2.1"},
    "us-east": {"value": "192.0.2.10"},
    "us-west": {"value": "192.0.2.20"},
    "eu-west": {"value": "192.0.2.30"},
    "eu-central": {"value": "192.0.2.31"},
    "apac-east": {"value": "192.0.2.40"}
  }
}
```

### AAAA Record with Geo Rules

```json
{
  "name": "www",
  "type": "AAAA",
  "ttl": 300,
  "geo_rules": {
    "default": {"value": "2001:db8::1"},
    "us-east": {"value": "2001:db8::10"},
    "eu-west": {"value": "2001:db8::30"},
    "apac-east": {"value": "2001:db8::40"}
  }
}
```

### CNAME with Geo Rules

```json
{
  "name": "api",
  "type": "CNAME",
  "ttl": 300,
  "geo_rules": {
    "default": {"value": "api-us.example.com"},
    "eu-west": {"value": "api-eu.example.com"},
    "apac-east": {"value": "api-ap.example.com"}
  }
}
```

## Weight Distribution

### Weighted Geo Rules

```json
{
  "name": "www",
  "type": "A",
  "geo_rules": {
    "us-east": {
      "value": "192.0.2.10",
      "weight": 60
    },
    "us-west": {
      "value": "192.0.2.20",
      "weight": 40
    }
  }
}
```

### Weight Calculation

When weights are specified, responses are distributed proportionally:

| Region | Weight | Percentage | Queries/1000 |
|--------|--------|-----------|--------------|
| US East | 60 | 60% | 600 |
| US West | 40 | 40% | 400 |

## Failover Configuration

### Geo Failover

```json
{
  "name": "www",
  "type": "A",
  "geo_rules": {
    "us-east": {
      "value": "192.0.2.10",
      "failover": {
        "enabled": true,
        "target": "192.0.2.100",
        "health_check": {
          "enabled": true,
          "interval": 30,
          "threshold": 3
        }
      }
    }
  }
}
```

### Failover Behavior

```
1. Primary target: 192.0.2.10
2. Health check fails 3 times
3. Automatic switch to failover: 192.0.2.100
4. Continue checking primary
5. When primary recovers, switch back (optional)
```

## EDNS Client Subnet

### How ECS Works

EDNS Client Subnet (ECS) passes the end-user's IP prefix to DNS resolvers for more accurate GeoIP decisions.

```
Without ECS:
User -> Local Resolver -> GeoDNS -> Returns nearest server (but resolver location used)

With ECS:
User (with subnet info) -> Local Resolver -> GeoDNS -> Returns nearest server (user location used)
```

### ECS Configuration

```php
// EDNS Client Subnet Configuration
$ecsConfig = [
    'enabled' => true,
    'client_subnet_ipv4' => '/24',      // IPv4 prefix length
    'client_subnet_ipv6' => '/56',       // IPv6 prefix length
    'cache_subnet_responses' => true,
    'privacy_mode' => false             // Don't log client IPs
];
```

## Web Interface

### Admin Panel

**Configuration > Geo DNS > Routing Rules**

```
+------------------------------------------------------------------+
| Geo DNS Rules for: example.com                         [Export] |
+------------------------------------------------------------------+
| Record: www.example.com | Type: A | TTL: 300                    |
|                                                                  |
| Rules Configuration:                                             |
| +--------------------------------------------------------------+ |
| | Region    | Value          | Weight | Failover | Actions     | |
| |-----------|----------------|--------|----------|-------------| |
| | Default   | 192.0.2.1     | -      | None     | [Edit][Del] | |
| | US East   | 192.0.2.10    | 100    | 192.0.2.100| [Edit][Del]| |
| | US West   | 192.0.2.20    | 100    | None     | [Edit][Del] | |
| | EU West   | 192.0.2.30    | 100    | None     | [Edit][Del] | |
| | EU Central| 192.0.2.31    | 100    | None     | [Edit][Del] | |
| | APAC East | 192.0.2.40   | 100    | None     | [Edit][Del] | |
| +--------------------------------------------------------------+ |
|                                                                  |
| [Add Rule] [Save All]                                            |
+------------------------------------------------------------------+
```

### Map Visualization

```
+------------------------------------------------------------------+
|  Geographic Distribution                                         |
+------------------------------------------------------------------+
|  [World Map with colored regions]                                |
|                                                                  |
|  Legend:                                                         |
|  [US East]   192.0.2.10  42% queries                             |
|  [US West]   192.0.2.20  23% queries                             |
|  [EU West]   192.0.2.30  18% queries                             |
|  [EU Central]192.0.2.31  12% queries                             |
|  [APAC East] 192.0.2.40  5% queries                              |
+------------------------------------------------------------------+
```

## API Reference

### Create Geo Rule

```http
POST /dns/zones/{zone}/geo-rules
```

### Update Geo Rule

```http
PUT /dns/zones/{zone}/geo-rules/{rule_id}
```

### Delete Geo Rule

```http
DELETE /dns/zones/{zone}/geo-rules/{rule_id}
```

### Test Geo Resolution

```http
GET /dns/test-geo-resolution
```

**Query Parameters:**
- `zone`: Zone name
- `name`: Record name
- `type`: Record type
- `ip`: Source IP to simulate
- `location`: Location to simulate (country code)

**Response:**

```json
{
  "query": {
    "zone": "example.com",
    "name": "www",
    "type": "A",
    "simulated_location": "US"
  },
  "result": {
    "value": "192.0.2.10",
    "region": "us-east",
    "rule_type": "country"
  }
}
```

## Performance Considerations

### TTL Recommendations

| Scenario | Recommended TTL | Reason |
|----------|-----------------|--------|
| Static geographic routing | 3600 (1 hour) | Reduce lookups |
| Active failover | 300 (5 minutes) | Faster switchover |
| Testing new rules | 60 (1 minute) | Quick iteration |
| Production stable | 1800 (30 minutes) | Balance |

### Cache Behavior

Geo DNS responses are cached based on:
1. Source IP / subnet
2. Record TTL
3. Resolver location

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong location | Outdated GeoIP | Update GeoIP database |
| All queries same region | ECS disabled | Enable ECS at resolver |
| Slow resolution | High TTL | Lower TTL for critical routes |
| Missing fallback | No default rule | Always add default |
| Failover not working | Health check misconfigured | Check health settings |

### Debug Tools

```bash
# Test geo resolution
whmcscli dns geo-test --zone=example.com --name=www --ip=8.8.8.8

# Check GeoIP database
whmcscli dns geoip-info --database=GeoLite2-City

# View geo rule statistics
whmcscli dns geo-stats --zone=example.com --period=24h
```

## See Also

- [Premium DNS Service](./whmcs-premium-dns-service.md)
- [Latency Routing](./whmcs-latency-routing.md)
- [Failover DNS](./whmcs-failover-dns.md)
