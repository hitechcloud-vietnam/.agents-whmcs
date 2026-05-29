# WHMCS SOA Record Configuration Documentation

## Overview

The Start of Authority (SOA) record contains administrative information about a DNS zone, including primary nameserver, administrator email, and timing parameters for zone transfers.

## SOA Record Structure

### Record Format

```
@  IN SOA  ns1.registrar.com.  admin.example.com.  (
        2024011501  ; Serial
        7200         ; Refresh
        3600         ; Retry
        1209600      ; Expire
        86400 )      ; Minimum TTL
```

### Field Definitions

| Field | Description | WHMCS Default |
|-------|-------------|----------------|
| MNAME | Primary nameserver | Set by provider |
| RNAME | Administrator email | admin@{domain} |
| SERIAL | Zone version number | Auto-incremented |
| REFRESH | Secondary refresh interval | 7200 (2 hours) |
| RETRY | Retry interval on failure | 3600 (1 hour) |
| EXPIRE | Time before secondary stops serving | 1209600 (14 days) |
| MINIMUM | Negative caching TTL | 86400 (24 hours) |

## Configuration

### Global SOA Settings

Navigate to: **Configuration > DNS Settings > SOA Configuration**

```php
// SOA Configuration
$soaConfig = [
    // Default values for new zones
    'default_refresh' => 7200,        // 2 hours
    'default_retry' => 3600,          // 1 hour
    'default_expire' => 1209600,      // 14 days
    'default_minimum' => 86400,       // 24 hours

    // Serial format
    'serial_format' => 'date_counter',  // YYYYMMDDNN
    'auto_increment' => true,

    // SOA record settings
    'primary_ns_pattern' => 'ns1.{registrar}.com',
    'admin_email_pattern' => 'admin@{domain}',

    // Validation
    'min_refresh' => 3600,            // 1 hour
    'max_refresh' => 86400,           // 24 hours
    'min_retry' => 1800,             // 30 minutes
    'max_retry' => 7200,             // 2 hours
    'min_expire' => 604800,          // 7 days
    'max_expire' => 2592000          // 30 days
];
```

### Per-Zone SOA Configuration

```php
// Per-zone SOA override
$zoneSoaConfig = [
    'zone' => 'example.com',
    'primary_ns' => 'ns1.custom.com',
    'admin_email' => 'dns@example.com',
    'refresh' => 14400,              // 4 hours
    'retry' => 7200,                 // 2 hours
    'expire' => 1814400,             // 21 days
    'minimum' => 43200               // 12 hours
];
```

## SOA Parameters

### Refresh Interval

Determines how often secondary nameservers check for zone updates.

```php
// Refresh interval recommendations
$refreshRecommendations = [
    'frequent_changes' => 3600,      // 1 hour - Active zones
    'standard' => 7200,              // 2 hours - Normal zones
    'stable' => 14400,              // 4 hours - Stable zones
    'minimal_changes' => 43200       // 12 hours - Rarely changed
];
```

**Best Practices:**
- Too short: Increased query load
- Too long: Slow propagation of changes
- Recommended: 2-4 hours for most zones

### Retry Interval

Time between refresh attempts when secondary cannot reach primary.

```php
// Retry should be less than refresh
$retryFormula = 'refresh / 2';  // Typical ratio
```

**Recommendations:**
- Should be less than Refresh
- Typical: 30 minutes to 2 hours
- Must be long enough to handle temporary outages

### Expire Interval

Time a secondary continues serving a zone if primary is unreachable.

```php
// Expire recommendations
$expireRecommendations = [
    'standard' => 1209600,          // 14 days - Default
    'extended' => 2419200,          // 28 days - Extended
    'critical' => 604800            // 7 days - Short for critical
];
```

**Recommendations:**
- Must be greater than Refresh + (Retry * 10)
- Longer = more resilient to outages
- Too short = zone may expire during extended outage

### Minimum TTL

Time resolvers cache negative responses (NXDOMAIN).

```php
// Minimum TTL recommendations
$minimumRecommendations = [
    'aggressive' => 300,             // 5 minutes - Rapid testing
    'standard' => 86400,             // 24 hours - Default
    'conservative' => 172800,        // 48 hours - Cache heavy
    'custom' => 3600                 // 1 hour - Custom option
];
```

## Serial Number

### Serial Format

```php
// Serial number format
$serialConfig = [
    'format' => 'date_counter',  // YYYYMMDDNN
    'counter_digits' => 2,
    'auto_increment' => true
];

// Example serial: 2024011501
// 2024 - Year
// 01   - Month
// 15   - Day
// 01   - Counter for multiple updates
```

### Serial Update Rules

1. Increment when zone changes
2. Serial must always increase
3. Secondary compares serial to detect changes
4. Format: YYYYMMDDNN or integer

### Manual Serial Update

```http
PATCH /dns/zones/{zone}/soa
```

**Request Body:**

```json
{
  "serial": 2024011502
}
```

## API Reference

### Get SOA Record

```http
GET /dns/zones/{zone}/soa
```

**Response:**

```json
{
  "zone": "example.com",
  "soa": {
    "primary_ns": "ns1.registrar.com",
    "admin_email": "admin@example.com",
    "serial": "2024011501",
    "refresh": 7200,
    "retry": 3600,
    "expire": 1209600,
    "minimum_ttl": 86400
  },
  "human_readable": {
    "refresh": "2 hours",
    "retry": "1 hour",
    "expire": "14 days",
    "minimum_ttl": "24 hours"
  }
}
```

### Update SOA Record

```http
PATCH /dns/zones/{zone}/soa
```

**Request Body:**

```json
{
  "refresh": 14400,
  "retry": 7200,
  "expire": 1814400,
  "minimum_ttl": 43200,
  "increment_serial": true
}
```

### Reset SOA to Defaults

```http
POST /dns/zones/{zone}/soa/reset
```

## Preset Configurations

### Standard Configuration

```json
{
  "primary_ns": "ns1.registrar.com",
  "admin_email": "admin@example.com",
  "refresh": 7200,
  "retry": 3600,
  "expire": 1209600,
  "minimum_ttl": 86400
}
```

### High Frequency Changes

```json
{
  "primary_ns": "ns1.registrar.com",
  "admin_email": "admin@example.com",
  "refresh": 1800,
  "retry": 900,
  "expire": 604800,
  "minimum_ttl": 300
}
```

### Stable Zones

```json
{
  "primary_ns": "ns1.registrar.com",
  "admin_email": "admin@example.com",
  "refresh": 14400,
  "retry": 7200,
  "expire": 2419200,
  "minimum_ttl": 172800
}
```

## Secondary DNS Integration

### SOA for Secondary DNS

When zone is configured for secondary DNS:

```php
// Secondary zone SOA
$secondarySoa = [
    'primary_ns' => 'ns1.primary-registrar.com',
    'refresh' => 7200,
    'retry' => 3600,
    'expire' => 1209600
];
```

### Zone Transfer Timing

```
Secondary Refresh Timer Expires
    |
    v
Query Primary for SOA
    |
    +-- Serial same --> Wait for Refresh
    |
    +-- Serial different --> Request AXFR/IXFR
```

## Troubleshooting

### Common SOA Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Zone not updating on secondary | Serial not incremented | Increment serial |
| Zone expired on secondary | Expire too short | Increase expire value |
| Slow propagation | Refresh too long | Reduce refresh interval |
| Negative caching issues | Minimum too long | Reduce minimum TTL |
| All zone changes ignored | Serial stuck | Force serial increment |

### SOA Validation

```php
// Validate SOA parameters
function validateSoa($soa) {
    $errors = [];

    // Refresh must be greater than Retry
    if ($soa['refresh'] <= $soa['retry']) {
        $errors[] = "Refresh must be greater than Retry";
    }

    // Expire should be much greater than Refresh
    if ($soa['expire'] < ($soa['refresh'] + ($soa['retry'] * 10))) {
        $errors[] = "Expire should be >= Refresh + (Retry * 10)";
    }

    // Check minimum TTL range
    if ($soa['minimum_ttl'] < 0 || $soa['minimum_ttl'] > 86400 * 7) {
        $errors[] = "Minimum TTL should be between 0 and 7 days";
    }

    return ['valid' => empty($errors), 'errors' => $errors];
}
```

### SOA Debug

```bash
# Get SOA record
dig @ns1.registrar.com SOA example.com +short

# Check serial progression
dig @ns1.registrar.com SOA example.com +noall +answer

# Verify zone serial match
whmcscli dns soa-check --zone=example.com
```

## Zone Templates

### SOA in Templates

```json
{
  "name": "standard-soa",
  "soa": {
    "primary_ns": "ns1.{provider}.com",
    "admin_email": "admin@{domain}",
    "refresh": 7200,
    "retry": 3600,
    "expire": 1209600,
    "minimum_ttl": 86400
  }
}
```

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [Secondary DNS](./whmcs-secondary-dns.md)
- [Records Management](./whmcs-records-management.md)
