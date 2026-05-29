# WHMCS Bulk DNS Operations Documentation

## Overview

Bulk DNS operations allow you to perform DNS changes on multiple domains simultaneously, significantly reducing administrative overhead.

## Bulk Operations

### Supported Operations

| Operation | Description | Use Case |
|-----------|-------------|----------|
| Add Records | Add same records to multiple zones | Batch configuration |
| Update Records | Modify records across zones | IP address changes |
| Delete Records | Remove records from multiple zones | Cleanup operations |
| Apply Templates | Apply DNS template to multiple zones | Standardization |
| Update Nameservers | Change NS records bulk | Registrar migration |
| Enable/Disable DNSSEC | Toggle DNSSEC | Security management |

## Configuration

### Enable Bulk Operations

Navigate to: **Configuration > Products/Services > Bulk DNS**

```php
// Bulk DNS Configuration
$bulkConfig = [
    'enabled' => true,
    'max_domains_per_operation' => 1000,
    'batch_size' => 50,
    'allow_async' => true,
    'async_timeout' => 3600,
    'require_confirmation' => true,
    'log_all_operations' => true,
    'auto_retry_failed' => false,
    'max_retries' => 3
];
```

## Bulk Add Records

### Add Same Record to Multiple Domains

```http
POST /dns/bulk/records/add
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net",
    "example.org"
  ],
  "record": {
    "name": "@",
    "type": "A",
    "value": "192.0.2.100",
    "ttl": 3600
  },
  "options": {
    "skip_existing": true,
    "continue_on_error": true
  }
}
```

**Response:**

```json
{
  "operation_id": "OP-12345",
  "total_domains": 3,
  "completed": 3,
  "failed": 0,
  "results": [
    {
      "domain": "example.com",
      "success": true,
      "record_id": "REC-001"
    },
    {
      "domain": "example.net",
      "success": true,
      "record_id": "REC-002"
    },
    {
      "domain": "example.org",
      "success": true,
      "record_id": "REC-003"
    }
  ]
}
```

### Add Multiple Records to Multiple Domains

```json
{
  "domains": [
    "example.com",
    "example.net"
  ],
  "records": [
    {"name": "@", "type": "A", "value": "192.0.2.1", "ttl": 3600},
    {"name": "www", "type": "CNAME", "value": "@", "ttl": 3600},
    {"name": "@", "type": "MX", "value": "mail.example.com", "priority": 10, "ttl": 3600}
  ]
}
```

## Bulk Update Records

### Update Record Value

```http
POST /dns/bulk/records/update
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net"
  ],
  "filter": {
    "name": "@",
    "type": "A"
  },
  "update": {
    "value": "192.0.2.200",
    "ttl": 7200
  }
}
```

### Replace Records

```json
{
  "domains": ["example.com"],
  "filter": {
    "type": "A"
  },
  "replace": [
    {"name": "@", "type": "A", "value": "192.0.2.200"},
    {"name": "www", "type": "A", "value": "192.0.2.200"}
  ]
}
```

## Bulk Delete Records

### Delete by Filter

```http
POST /dns/bulk/records/delete
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net",
    "example.org"
  ],
  "filter": {
    "name": "old-subdomain",
    "type": "A"
  },
  "confirm": true
}
```

### Delete All Records of Type

```json
{
  "domains": ["example.com"],
  "filter": {
    "type": "MX"
  },
  "confirm": true,
  "warning": "This will remove all MX records"
}
```

## Bulk Template Application

### Apply Template to Multiple Domains

```http
POST /dns/bulk/apply-template
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net",
    "example.org",
    "test.com"
  ],
  "template": "standard-web",
  "variables": {
    "server_ip": "192.0.2.1"
  },
  "options": {
    "replace_existing": false,
    "skip_existing": true
  }
}
```

### Bulk Import Zones with Template

```json
{
  "domains": ["new1.com", "new2.com", "new3.com"],
  "template": "new-domain-setup",
  "variables": {
    "primary_ip": "192.0.2.1",
    "secondary_ip": "192.0.2.2",
    "mail_server": "mail"
  },
  "nameservers": {
    "ns1": "ns1.your-dns.com",
    "ns2": "ns2.your-dns.com"
  }
}
```

## Bulk Nameserver Updates

### Update Nameservers

```http
POST /dns/bulk/nameservers
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net",
    "example.org"
  ],
  "nameservers": [
    "ns1.new-registrar.com",
    "ns2.new-registrar.com"
  ]
}
```

### Verify Before Update

```json
{
  "domains": ["example.com"],
  "nameservers": [
    "ns1.new-registrar.com",
    "ns2.new-registrar.com"
  ],
  "verify_only": true
}
```

**Response:**

```json
{
  "verified": true,
  "domains": [
    {
      "domain": "example.com",
      "current_nameservers": ["ns1.old-registrar.com", "ns2.old-registrar.com"],
      "new_nameservers": ["ns1.new-registrar.com", "ns2.new-registrar.com"],
      "will_change": true
    }
  ]
}
```

## Bulk DNSSEC Operations

### Enable DNSSEC

```http
POST /dns/bulk/dnssec/enable
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net",
    "example.org"
  ]
}
```

### Disable DNSSEC

```http
POST /dns/bulk/dnssec/disable
```

## Domain Filtering

### Filter by Criteria

```http
POST /dns/bulk/filter
```

**Request Body:**

```json
{
  "filters": {
    "registrar": "enom",
    "status": ["active", "expired"],
    "tld": ["com", "net", "org"],
    "expiry_before": "2025-01-01",
    "has_dnssec": false,
    "has_records": true,
    "record_type": "MX"
  },
  "limit": 100
}
```

## Async Operations

### Start Async Bulk Operation

```http
POST /dns/bulk/async/start
```

**Request Body:**

```json
{
  "operation": "add_records",
  "domains": ["domain1.com", "domain2.com", ...],
  "record": {
    "name": "@",
    "type": "TXT",
    "value": "verification=abc123"
  }
}
```

**Response:**

```json
{
  "operation_id": "ASYNC-12345",
  "status": "started",
  "estimated_duration": "5 minutes",
  "status_url": "/dns/bulk/async/ASYNC-12345/status"
}
```

### Check Operation Status

```http
GET /dns/bulk/async/{operation_id}/status
```

**Response:**

```json
{
  "operation_id": "ASYNC-12345",
  "operation": "add_records",
  "status": "in_progress",
  "progress": {
    "total": 1000,
    "completed": 450,
    "failed": 2,
    "percentage": 45
  },
  "started_at": "2024-01-15T10:00:00Z",
  "estimated_completion": "2024-01-15T10:05:00Z"
}
```

### Cancel Async Operation

```http
POST /dns/bulk/async/{operation_id}/cancel
```

## Operation History

### List Past Operations

```http
GET /dns/bulk/history
```

**Query Parameters:**
- `status`: completed, failed, cancelled
- `operation`: add_records, update, delete, etc.
- `from`: Start date
- `to`: End date
- `limit`: Results limit

**Response:**

```json
{
  "operations": [
    {
      "id": "OP-12345",
      "operation": "add_records",
      "status": "completed",
      "total_domains": 100,
      "successful": 98,
      "failed": 2,
      "created_at": "2024-01-15T10:00:00Z",
      "completed_at": "2024-01-15T10:05:00Z"
    }
  ]
}
```

### Get Operation Details

```http
GET /dns/bulk/history/{operation_id}
```

## CSV Import/Export

### Bulk Import via CSV

```http
POST /dns/bulk/import/csv
```

**CSV Format:**

```csv
domain,record_type,record_name,value,ttl,priority
example.com,A,@,192.0.2.1,3600,
example.com,CNAME,www,@,3600,
example.com,MX,@,mail.example.com,3600,10
example.net,A,@,192.0.2.1,3600,
```

**Request:**

```json
{
  "file": "base64_encoded_csv",
  "options": {
    "skip_header": true,
    "continue_on_error": true
  }
}
```

### Export Domains to CSV

```http
GET /dns/bulk/export
```

**Query Parameters:**
- `domains`: Comma-separated list or filter JSON
- `format`: csv, json
- `fields`: domain,type,name,value,ttl,priority

## Web Interface

### Bulk Operations Panel

**Configuration > Bulk DNS > Operations**

```
+------------------------------------------------------------------+
|  Bulk DNS Operations                                              |
+------------------------------------------------------------------+
|                                                                  |
|  [Add Records] [Update Records] [Delete Records]                 |
|  [Apply Template] [Update Nameservers] [DNSSEC]                   |
|                                                                  |
|  Select Domains:                                                 |
|  +--------------------------------------------------------------+|
|  | [x] example.com                                             ||
|  | [x] example.net                                             ||
|  | [x] example.org                                             ||
|  | [ ] domain4.com                                             ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Or Filter Domains:                                              |
|  Registrar: [All____________] TLD: [All______]                   |
|  Status:   [Active_________]                                     |
|                                                                  |
|  [Select Filtered (150 domains)] [Apply]                         |
+------------------------------------------------------------------+
```

## Validation Rules

### Record Validation

| Rule | Description |
|------|-------------|
| Max records per zone | 1000 |
| Max bulk domains | 1000 |
| Valid record types | A, AAAA, CNAME, MX, TXT, etc. |
| IP validation | Strict IPv4/IPv6 format |
| TTL range | 60 - 86400 seconds |

### Domain Validation

| Rule | Description |
|------|-------------|
| Domain exists | Must be registered in WHMCS |
| DNS enabled | Domain must have DNS enabled |
| No pending operations | Domain cannot be in another bulk op |

## Error Handling

### Error Codes

| Code | Error | Resolution |
|------|-------|------------|
| BULK_001 | Domain not found | Check domain names |
| BULK_002 | Invalid record | Validate record format |
| BULK_003 | Record limit exceeded | Remove some records |
| BULK_004 | Duplicate record | Skip or update existing |
| BULK_005 | Operation cancelled | Restart operation |
| BULK_006 | Timeout | Retry with smaller batch |

### Retry Failed Items

```http
POST /dns/bulk/{operation_id}/retry-failed
```

## Performance

### Optimization Tips

1. **Use async for large operations** - >100 domains should use async
2. **Batch appropriately** - 50-100 domains per batch
3. **Filter before bulk** - Narrow down to only affected domains
4. **Off-peak scheduling** - Schedule large operations off-peak
5. **Verify before execute** - Use verify_only to check first

### Rate Limits

| Plan | Bulk Operations/Hour | Max Domains/Operation |
|------|---------------------|----------------------|
| Standard | 10 | 100 |
| Professional | 50 | 500 |
| Enterprise | 200 | 1000 |

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [DNS Templates](./whmcs-dns-templates.md)
- [DNS Management API](./whmcs-dns-management-api.md)
