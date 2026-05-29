# WHMCS Secondary DNS Documentation

## Overview

Secondary DNS provides redundant nameserver infrastructure for improved reliability, faster propagation, and distributed query handling.

## How Secondary DNS Works

### Architecture

```
Primary Nameserver          Secondary Nameservers
+----------------+         +----------------+
| Zone: example  |   AXFR  | Zone: example  |
| 192.0.2.1      |------->| 192.0.3.1      |
| (Authoritative) |   IXFR  | (Secondary)   |
|                |<-------| (Authoritative)|
+----------------+         +----------------+
                                   |
                                   v
                          +----------------+
                          | Query Traffic  |
                          | Distributed    |
                          +----------------+
```

### Zone Transfer Types

| Type | Description | Delta |
|------|-------------|-------|
| AXFR | Full zone transfer | Entire zone |
| IXFR | Incremental zone transfer | Only changes |
| NOTIFY | Primary notifies secondary of changes | Signal only |

## Configuration

### Enable Secondary DNS

Navigate to: **Configuration > Products/Services > Secondary DNS**

```php
// Secondary DNS Configuration
$secondaryDnsConfig = [
    'enabled' => true,
    'allow_zone_transfer' => true,
    'require_tsig' => true,
    'axfr_enabled' => true,
    'ixfr_enabled' => true,
    'notify_enabled' => true,
    'refresh_interval' => 7200,         // 2 hours
    'retry_interval' => 3600,          // 1 hour
    'expire_interval' => 1209600,      // 14 days
    'max_transfer_size' => 10485760     // 10MB
];
```

### Zone Transfer Settings

```php
// Zone Transfer Configuration
$transferConfig = [
    'allowed_ips' => [
        '192.0.3.0/24',      // Secondary NS network
        '192.0.4.0/24'       // Backup network
    ],
    'tsig_required' => true,
    'tsig_algorithm' => 'hmac-sha256',
    'axfr_rate_limit' => 10,           // per minute
    'ixfr_enabled' => true,
    'max_connections' => 5
];
```

## Secondary Nameserver Setup

### Register Secondary NS

```http
POST /dns/secondary/servers
```

**Request Body:**

```json
{
  "hostname": "ns2.backup-dns.com",
  "ipv4": "192.0.3.1",
  "ipv6": "2001:db8:3::1",
  "tsig_key": {
    "name": "secondary-dns-key",
    "algorithm": "hmac-sha256",
    "secret": "base64_encoded_secret"
  },
  "capabilities": ["axfr", "ixfr", "notify"]
}
```

### Configure Zone for Secondary

```http
POST /dns/zones/{zone}/secondary
```

**Request Body:**

```json
{
  "primary_ns": "ns1.primary.com",
  "primary_ip": "192.0.2.1",
  "transfer_type": "axfr",
  "tsig_enabled": true,
  "tsig_key_name": "zone-transfer-key",
  "notify_enabled": true,
  "refresh_interval": 7200,
  "retry_interval": 3600
}
```

## Primary Nameserver Configuration

### Enable AXFR on Primary

```php
// Primary Nameserver Config
$primaryConfig = [
    'allow_transfer' => [
        '192.0.3.1',    // Secondary NS 1
        '192.0.3.2',    // Secondary NS 2
        '192.0.4.1'     // Backup Secondary
    ],
    'tsig_keys' => [
        'secondary-dns-key' => [
            'algorithm' => 'hmac-sha256',
            'secret' => 'base64_secret'
        ]
    ],
    'notify_settings' => [
        'enabled' => true,
        'also_notify' => ['192.0.3.1', '192.0.3.2']
    ]
];
```

### BIND Configuration Example

```conf
// named.conf.local

zone "example.com" {
    type master;
    file "/etc/bind/zones/example.com.db";
    allow-transfer {
        192.0.3.1;
        192.0.3.2;
        key "secondary-dns-key";
    };
    also-notify {
        192.0.3.1;
        192.0.3.2;
    };
};

key "secondary-dns-key" {
    algorithm hmac-sha256;
    secret "base64_secret_here";
};
```

## Zone Transfer Process

### AXFR (Full Transfer)

```
1. Secondary checks refresh timer (default: 2 hours)
2. Secondary sends SOA query to Primary
3. Primary responds with current serial
4. Secondary compares serial
5. If different, Secondary requests AXFR
6. Primary sends full zone
7. Secondary updates zone file
8. Zone is now in sync
```

### IXFR (Incremental Transfer)

```
1. Secondary checks refresh timer
2. Secondary sends SOA query
3. Primary responds with new serial
4. Secondary requests IXFR with current serial
5. Primary determines delta needed
6. Primary sends incremental changes
7. Secondary applies changes
8. Zone is now in sync
```

### NOTIFY Mechanism

```
1. Zone is updated on Primary
2. Primary increments serial
3. Primary sends NOTIFY to Secondaries
4. Secondary immediately requests SOA
5. Secondary requests transfer
6. Zone synced
```

## TSIG Authentication

### Generate TSIG Key

```bash
# Generate TSIG key using dnssec-keygen
dnssec-keygen -a HMAC-SHA256 -b 256 -n HOST secondary-dns-key
```

### TSIG Configuration

```php
// TSIG Key Setup
$tsigConfig = [
    'name' => 'secondary-dns-key',
    'algorithm' => 'hmac-sha256',
    'secret' => 'generated_base64_secret'
];
```

### Verify TSIG

```bash
# Test zone transfer with TSIG
dig @192.0.2.1 axfr example.com -y hmac-sha256:secondary-dns-key:base64_secret
```

## Monitoring

### Transfer Status

```http
GET /dns/secondary/status/{zone}
```

**Response:**

```json
{
  "zone": "example.com",
  "primary": {
    "hostname": "ns1.primary.com",
    "ip": "192.0.2.1",
    "serial": 2024011501
  },
  "secondaries": [
    {
      "id": "sec-1",
      "hostname": "ns2.backup-dns.com",
      "ip": "192.0.3.1",
      "status": "in_sync",
      "last_transfer": "2024-01-15T10:00:00Z",
      "serial": 2024011501,
      "queries_today": 456789
    },
    {
      "id": "sec-2",
      "hostname": "ns3.backup-dns.com",
      "ip": "192.0.3.2",
      "status": "syncing",
      "last_transfer": "2024-01-15T09:30:00Z",
      "serial": 2024011500,
      "progress": "75%"
    }
  ],
  "transfer_stats": {
    "axfr_today": 5,
    "ixfr_today": 45,
    "failed_transfers": 0,
    "avg_transfer_time_ms": 1250
  }
}
```

### Transfer History

```http
GET /dns/secondary/transfers/{zone}/history
```

```json
{
  "zone": "example.com",
  "transfers": [
    {
      "id": "XFR-12345",
      "type": "AXFR",
      "secondary": "192.0.3.1",
      "started_at": "2024-01-15T10:00:00Z",
      "completed_at": "2024-01-15T10:00:15Z",
      "records_transferred": 1500,
      "bytes_transferred": 125000,
      "status": "success"
    },
    {
      "id": "XFR-12344",
      "type": "IXFR",
      "secondary": "192.0.3.1",
      "started_at": "2024-01-15T08:00:00Z",
      "completed_at": "2024-01-15T08:00:02Z",
      "changes": 3,
      "status": "success"
    }
  ]
}
```

## Secondary DNS Management

### List Secondary Zones

```http
GET /dns/secondary/zones
```

### Sync Zone Now

```http
POST /dns/secondary/zones/{zone}/sync
```

**Response:**

```json
{
  "zone": "example.com",
  "sync_initiated": true,
  "estimated_completion": "2024-01-15T10:05:00Z",
  "secondaries_notified": ["192.0.3.1", "192.0.3.2"]
}
```

### Remove Secondary

```http
DELETE /dns/secondary/zones/{zone}/secondaries/{secondary_id}
```

## Query Distribution

### Load Balancing Configuration

```php
// Query Distribution
$queryConfig = [
    'strategy' => 'round_robin',
    'primary_weight' => 100,
    'secondary_weight' => 50,
    'health_check_enabled' => true,
    'fallback_to_primary' => true
];
```

### DNS Round Robin

With multiple secondaries, configure round-robin:

```json
{
  "zone": "example.com",
  "nameservers": [
    {"host": "ns1.primary.com", "ip": "192.0.2.1", "weight": 100},
    {"host": "ns2.secondary.com", "ip": "192.0.3.1", "weight": 50},
    {"host": "ns3.secondary.com", "ip": "192.0.3.2", "weight": 50}
  ]
}
```

## Security

### IP Whitelisting

```php
// Access Control
$accessControl = [
    'allow_transfer_from' => [
        '192.0.3.0/24'    // Secondary NS network
    ],
    'allow_query_from' => 'any',
    'rate_limit' => [
        'enabled' => true,
        'queries_per_second' => 1000
    ]
];
```

### TSIG Best Practices

1. Use strong algorithms (HMAC-SHA256 or better)
2. Rotate keys regularly (every 90 days)
3. Store keys securely (encrypted at rest)
4. Use separate keys per zone when possible
5. Monitor for transfer attempts without keys

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Transfer not working | Firewall blocking port 53 | Open UDP/TCP 53 |
| TSIG mismatch | Key mismatch | Verify keys on both sides |
| Stale zone data | Refresh interval too long | Lower refresh interval |
| AXFR failing | Serial not incrementing | Check zone update process |
| Notify not received | Notify ACL misconfigured | Check also-notify list |

### Debug Commands

```bash
# Check zone transfer status
whmcscli dns secondary status --zone=example.com

# Force zone transfer
whmcscli dns secondary transfer --zone=example.com --secondary=192.0.3.1

# Test AXFR
dig @192.0.2.1 axfr example.com

# Check SOA serial
dig @192.0.2.1 SOA example.com +short

# Verify TSIG
dig @192.0.2.1 AXFR example.com -y hmac-sha256:secondary-dns-key:secret
```

### Log Analysis

```
# Zone transfer log
2024-01-15 10:00:00 - AXFR from 192.0.3.1 for example.com - SUCCESS (1500 records)
2024-01-15 08:00:00 - IXFR from 192.0.3.1 for example.com - SUCCESS (3 changes)
2024-01-15 06:00:00 - AXFR from 192.0.3.2 for example.com - FAILED: TSIG error
```

## Web Interface

### Secondary DNS Panel

**Configuration > Secondary DNS > Manage**

```
+------------------------------------------------------------------+
|  Secondary DNS Management                                         |
+------------------------------------------------------------------+
|                                                                  |
|  Primary Zones: 25 | Secondary Zones: 50 | Transfer Rate: 95%    |
|                                                                  |
|  Zone: example.com                                               |
|  Primary: ns1.primary.com (192.0.2.1)                            |
|  Serial: 2024011501                                              |
|                                                                  |
|  Secondaries:                                                    |
|  +--------------------------------------------------------------+|
|  | Hostname           | IP          | Status    | Last Sync    ||
|  |--------------------|-------------|-----------|--------------||
|  | ns2.backup-dns.com | 192.0.3.1   | In Sync   | 10:00:00     ||
|  | ns3.backup-dns.com | 192.0.3.2   | In Sync   | 09:55:00     ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Add Secondary] [Sync Now] [View Logs] [Settings]              |
+------------------------------------------------------------------+
```

## Performance Optimization

### Recommendations

| Setting | Recommended Value | Rationale |
|---------|-------------------|-----------|
| Refresh Interval | 7200 (2 hours) | Balance between freshness and load |
| Retry Interval | 3600 (1 hour) | Retry before refresh |
| Expire Interval | 1209600 (14 days) | Keep serving if primary down |
| Enable IXFR | Yes | Reduce transfer size |
| TSIG | Required | Security |

### Monitoring Metrics

- Transfer success rate
- Time since last successful transfer
- Zone serial consistency across secondaries
- Query distribution across nameservers
- Transfer latency

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [SOA Records](./whmcs-soa-records.md)
- [Premium DNS](./whmcs-premium-dns-service.md)
