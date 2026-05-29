# WHMCS DNSSEC API Documentation

## Overview

DNSSEC (DNS Security Extensions) adds cryptographic signatures to DNS records, enabling resolvers to verify the authenticity of DNS data.

## How DNSSEC Works

### Chain of Trust

```
Root DNSKEY
    |
    v
TLD DNSKEY (e.g., .com)
    |
    v
Domain DNSKEY + RRSIG
    |
    v
Verified Response
```

### Key Types

| Key Type | Purpose | Rollover Frequency |
|----------|---------|---------------------|
| KSK (Key Signing Key) | Signs ZSK and DNSKEY set | 1-2 years |
| ZSK (Zone Signing Key) | Signs all other records | 1-3 months |

## Configuration

### Enable DNSSEC

Navigate to: **Configuration > Products/Services > DNSSEC**

```php
// DNSSEC Configuration
$dnssecConfig = [
    'enabled' => true,
    'auto_sign' => true,
    'algorithm' => 13,              // ECDSAP256SHA256
    'ksk_bits' => 256,
    'zsk_bits' => 256,
    'ksk_rollover_interval' => 365, // days
    'zsk_rollover_interval' => 90,  // days
    'signing_method' => 'nsec3'     // nsec or nsec3
];
```

## API Endpoints

### Enable DNSSEC for Zone

```http
POST /dns/zones/{zone}/dnssec/enable
```

**Response:**

```json
{
  "success": true,
  "zone": "example.com",
  "dnssec_enabled": true,
  "keys": [
    {
      "id": "KSK-001",
      "type": "KSK",
      "algorithm": 13,
      "key_tag": 12345,
      "flags": 257,
      "public_key": "AwEAAdR82...",
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "ZSK-001",
      "type": "ZSK",
      "algorithm": 13,
      "key_tag": 67890,
      "flags": 256,
      "public_key": "AwEAAdR83...",
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "ds_records": [
    {
      "key_tag": 12345,
      "algorithm": 13,
      "digest_type": 2,
      "digest": "AABBCCDDEEFF00112233...",
      "record": "example.com. 3600 IN DS 12345 13 2 AABBCC..."
    }
  ]
}
```

### Get DNSSEC Status

```http
GET /dns/zones/{zone}/dnssec
```

**Response:**

```json
{
  "zone": "example.com",
  "dnssec_enabled": true,
  "signed_at": "2024-01-01T00:00:00Z",
  "keys": [
    {
      "id": "KSK-001",
      "type": "KSK",
      "algorithm": 13,
      "key_tag": 12345,
      "key_type": "active",
      "public_key": "AwEAAdR82...",
      "created_at": "2024-01-01T00:00:00Z",
      "expires_at": "2025-01-01T00:00:00Z"
    },
    {
      "id": "ZSK-001",
      "type": "ZSK",
      "algorithm": 13,
      "key_tag": 67890,
      "key_type": "active",
      "public_key": "AwEAAdR83...",
      "created_at": "2024-01-01T00:00:00Z",
      "expires_at": "2024-04-01T00:00:00Z"
    }
  ],
  "ds_records": [
    {
      "key_tag": 12345,
      "algorithm": 13,
      "digest_type": 2,
      "digest": "AABBCCDDEEFF...",
      "record": "example.com. 3600 IN DS 12345 13 2 AABBCC..."
    }
  ],
  "validation": {
    "chain_valid": true,
    "last_validated": "2024-01-15T10:00:00Z"
  }
}
```

### Disable DNSSEC

```http
POST /dns/zones/{zone}/dnssec/disable
```

**Request Body:**

```json
{
  "remove_ds_records": true,
  "confirm": true
}
```

**Response:**

```json
{
  "success": true,
  "zone": "example.com",
  "dnssec_enabled": false,
  "disabled_at": "2024-01-15T10:30:00Z"
}
```

## Key Management

### List Keys

```http
GET /dns/zones/{zone}/dnssec/keys
```

**Response:**

```json
{
  "zone": "example.com",
  "keys": [
    {
      "id": "KSK-001",
      "type": "KSK",
      "algorithm": 13,
      "key_tag": 12345,
      "flags": 257,
      "public_key": "AwEAAdR82...",
      "private_key_available": true,
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z",
      "rollover_at": "2025-01-01T00:00:00Z",
      "in_dnskey": true,
      "signed_zsk": true
    },
    {
      "id": "ZSK-001",
      "type": "ZSK",
      "algorithm": 13,
      "key_tag": 67890,
      "flags": 256,
      "public_key": "AwEAAdR83...",
      "private_key_available": true,
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z",
      "rollover_at": "2024-04-01T00:00:00Z",
      "in_dnskey": true,
      "signs_records": true
    }
  ]
}
```

### Generate New Key

```http
POST /dns/zones/{zone}/dnssec/keys
```

**Request Body:**

```json
{
  "type": "KSK",
  "algorithm": 13,
  "bits": 256,
  "activate": false
}
```

**Response:**

```json
{
  "success": true,
  "key": {
    "id": "KSK-002",
    "type": "KSK",
    "algorithm": 13,
    "key_tag": 54321,
    "flags": 257,
    "public_key": "AwEAAdR84...",
    "status": "pending_activation",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

### Activate Key

```http
POST /dns/zones/{zone}/dnssec/keys/{key_id}/activate
```

### Deactivate Key

```http
POST /dns/zones/{zone}/dnssec/keys/{key_id}/deactivate
```

### Delete Key

```http
DELETE /dns/zones/{zone}/dnssec/keys/{key_id}
```

## Key Rollover

### Schedule Rollover

```http
POST /dns/zones/{zone}/dnssec/keys/{key_id}/rollover
```

**Request Body:**

```json
{
  "scheduled_date": "2024-02-01",
  "method": "double_signature"
}
```

### Rollover Methods

| Method | Description | Downtime Risk |
|--------|-------------|---------------|
| Double Signature | Both old and new keys active during transition | None |
| Pre-Publication | New key published before old key removed | None |
| Hard Rollover | Immediate swap | Risk of validation failure |

### Rollover Status

```http
GET /dns/zones/{zone}/dnssec/keys/{key_id}/rollover-status
```

**Response:**

```json
{
  "key_id": "KSK-002",
  "rollover_status": "in_progress",
  "phase": "new_key_published",
  "started_at": "2024-02-01T00:00:00Z",
  "estimated_completion": "2024-02-08T00:00:00Z",
  "steps": [
    {"step": "generate_key", "status": "completed"},
    {"step": "publish_dnskey", "status": "completed"},
    {"step": "propagate_ds", "status": "in_progress"},
    {"step": "activate_key", "status": "pending"},
    {"step": "remove_old_key", "status": "pending"}
  ]
}
```

## DS Records

### Get DS Records

```http
GET /dns/zones/{zone}/dnssec/ds
```

**Response:**

```json
{
  "zone": "example.com",
  "ds_records": [
    {
      "key_tag": 12345,
      "algorithm": 13,
      "digest_type": 2,
      "digest": "AABBCCDDEEFF001122...",
      "record": "example.com. 3600 IN DS 12345 13 2 AABBCCDDEEFF001122...",
      "status": "published",
      "parent_published": true
    }
  ]
}
```

### Add DS Record

```http
POST /dns/zones/{zone}/dnssec/ds
```

**Request Body:**

```json
{
  "key_id": "KSK-001",
  "algorithm": 13,
  "digest_type": 2,
  "digest": "AABBCCDDEEFF001122..."
}
```

### DS Record Formats

```
# Standard DS Record Format
example.com. 3600 IN DS <key-tag> <algorithm> <digest-type> <digest>

# Example
example.com. 3600 IN DS 12345 13 2 AABBCCDDEEFF00112233...

# Multi-DS (for multiple algorithms)
example.com. 3600 IN DS 12345 13 2 AABBCC...
example.com. 3600 IN DS 12345 14 2 DDEEFF...
```

### Digest Types

| Type | Algorithm | Status |
|------|-----------|--------|
| 1 | SHA-1 | Deprecated |
| 2 | SHA-256 | Recommended |
| 4 | SHA-384 | Recommended |

### Algorithm Types

| Algorithm | Name | Status |
|-----------|------|--------|
| 5 | RSASHA1 | Deprecated |
| 7 | RSASHA1-NSEC3-SHA1 | Deprecated |
| 8 | RSASHA256 | Deprecated |
| 10 | RSASHA512 | Deprecated |
| 13 | ECDSAP256SHA256 | Recommended |
| 14 | ECDSAP384SHA384 | Recommended |

## DNSKEY Records

### Get DNSKEY Set

```http
GET /dns/zones/{zone}/dnssec/dnskey
```

**Response:**

```json
{
  "zone": "example.com",
  "dnskey_set": [
    {
      "flags": 257,
      "protocol": 3,
      "algorithm": 13,
      "key_tag": 12345,
      "public_key": "AwEAAdR82..."
    },
    {
      "flags": 256,
      "protocol": 3,
      "algorithm": 13,
      "key_tag": 67890,
      "public_key": "AwEAAdR83..."
    }
  ],
  "rrsig_count": 15,
  "last_signed": "2024-01-15T10:00:00Z"
}
```

## Validation

### Check DNSSEC Validation

```http
GET /dns/zones/{zone}/dnssec/validate
```

**Response:**

```json
{
  "zone": "example.com",
  "validation": {
    "chain_valid": true,
    "validated_at": "2024-01-15T10:30:00Z",
    "checks": [
      {"check": "dnskey_present", "status": "pass"},
      {"check": "keys_valid", "status": "pass"},
      {"check": "signatures_valid", "status": "pass"},
      {"check": "ds_at_parent", "status": "pass"},
      {"check": "chain_of_trust", "status": "pass"}
    ]
  },
  "dnsviz_result": "https://dnsviz.net/d/example.com/dnssec/"
}
```

### Test DNSSEC Propagation

```http
GET /dns/zones/{zone}/dnssec/propagation
```

**Response:**

```json
{
  "zone": "example.com",
  "propagation_status": {
    "dnskey_propagated": true,
    "ds_propagated": true,
    "global_spread": 95
  },
  "dns_servers_tested": [
    {"server": "8.8.8.8", "dnskey_valid": true, "ds_valid": true},
    {"server": "1.1.1.1", "dnskey_valid": true, "ds_valid": true},
    {"server": "9.9.9.9", "dnskey_valid": true, "ds_valid": true}
  ]
}
```

## Monitoring

### DNSSEC Health

```http
GET /dns/zones/{zone}/dnssec/health
```

**Response:**

```json
{
  "zone": "example.com",
  "health": {
    "overall_status": "healthy",
    "issues": [],
    "warnings": [
      {
        "type": "key_expiring",
        "key_id": "ZSK-001",
        "days_until_expiry": 30
      }
    ]
  },
  "statistics": {
    "signature_count": 150,
    "average_signature_ttl": 3600,
    "keys_in_rotation": 1
  }
}
```

### DNSSEC Events

```http
GET /dns/zones/{zone}/dnssec/events
```

## Automated Management

### Enable Auto-Rollover

```http
POST /dns/zones/{zone}/dnssec/auto-rollover
```

**Request Body:**

```json
{
  "enabled": true,
  "ksk_rollover_interval": 365,
  "zsk_rollover_interval": 90,
  "notify_before_expiry_days": 7
}
```

### Rollover Alerts

```php
// Rollover Alert Configuration
$rolloverAlerts = [
    'enabled' => true,
    'alert_days_before' => [30, 14, 7, 3, 1],
    'channels' => ['email', 'webhook'],
    'email_recipients' => ['admin@example.com']
];
```

## Registrar Integration

### Publish DS to Registry

```http
POST /dns/zones/{zone}/dnssec/ds/publish
```

**Request Body:**

```json
{
  "registrar": "enom",
  "ds_records": [
    "example.com. 3600 IN DS 12345 13 2 AABBCC..."
  ]
}
```

### Check DS Publication Status

```http
GET /dns/zones/{zone}/dnssec/ds/status
```

**Response:**

```json
{
  "zone": "example.com",
  "ds_records": [
    {
      "key_tag": 12345,
      "status": "published",
      "published_at": "2024-01-15T10:30:00Z",
      "registrar": "enom"
    }
  ],
  "registry_verification": {
    "whois_check": "published",
    "dns_check": "published"
  }
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Validation failing | DS not at parent | Add DS to registrar |
| Expired signatures | Keys not rolled | Trigger manual rollover |
| Algorithm mismatch | Unsupported algorithm | Use ECDSAP256SHA256 |
| Chain broken | Missing parent DS | Check registry DS |
| NSEC3 issues | Broken NSEC3 chain | Regenerate NSEC3 chain |

### Debug Commands

```bash
# Check DNSSEC status
whmcscli dnssec status --zone=example.com

# Verify DNSSEC chain
whmcscli dnssec verify --zone=example.com

# Check DS at parent
whmcscli dnssec ds-check --zone=example.com

# Force resign zone
whmcscli dnssec resign --zone=example.com --force
```

### DNSSEC Validators

| Tool | URL |
|------|-----|
| DNSViz | https://dnsviz.net/ |
| VeriDNS | https://veridns.com/ |
| dnssec-analyzer | https://dnssec-analyzer.verisignlabs.com/ |

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [SOA Records](./whmcs-soa-records.md)
- [Premium DNS Service](./whmcs-premium-dns-service.md)
