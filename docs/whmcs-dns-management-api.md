# WHMCS DNS Management API Documentation

## Overview

The WHMCS DNS Management API provides comprehensive programmatic control over DNS records, zones, and settings. This API enables integration with DNS providers and custom DNS management solutions.

## Authentication

### API Key Authentication

```http
Authorization: Bearer your_api_key
Content-Type: application/json
```

### OAuth 2.0 Authentication

```http
Authorization: Bearer access_token
```

## Base URL

```
Production: https://api.whmcs.com/v1/dns
Sandbox:    https://sandbox-api.whmcs.com/v1/dns
```

## DNS Record Types

### Supported Record Types

| Type | Code | Description | Use Case |
|------|------|-------------|----------|
| A | 1 | IPv4 Address | Point domain to server IP |
| AAAA | 28 | IPv6 Address | Point domain to IPv6 |
| CNAME | 5 | Canonical Name | Alias to another domain |
| MX | 15 | Mail Exchange | Email routing |
| TXT | 16 | Text Record | SPF, DKIM, verification |
| NS | 2 | Nameserver | Delegate subdomain |
| SRV | 33 | Service Record | Service location |
| PTR | 12 | Pointer | Reverse DNS |
| CAA | 257 | CA Authorization | SSL certificate issuers |
| SOA | 6 | Start of Authority | Zone configuration |

## Endpoints

### Zone Management

#### Get Zone Information

```http
GET /zones/{zone}
```

**Response:**

```json
{
  "zone": "example.com",
  "zone_id": "ZONE-12345",
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "ttl": 3600,
  "refresh": 7200,
  "retry": 3600,
  "expire": 1209600,
  "minimum_ttl": 86400,
  "nameservers": [
    "ns1.dnsprovider.com",
    "ns2.dnsprovider.com"
  ],
  "record_count": 15
}
```

#### Create Zone

```http
POST /zones
```

**Request Body:**

```json
{
  "domain": "example.com",
  "ttl": 3600,
  "nameservers": [
    "ns1.dnsprovider.com",
    "ns2.dnsprovider.com"
  ],
  "template": "default"
}
```

#### Update Zone Settings

```http
PATCH /zones/{zone}
```

**Request Body:**

```json
{
  "ttl": 7200,
  "refresh": 14400,
  "retry": 1800,
  "expire": 604800,
  "minimum_ttl": 43200
}
```

#### Delete Zone

```http
DELETE /zones/{zone}
```

**Response:**

```json
{
  "success": true,
  "zone": "example.com",
  "deleted_at": "2024-01-15T10:30:00Z"
}
```

### Record Management

#### List Records

```http
GET /zones/{zone}/records
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| type | string | Filter by record type |
| name | string | Filter by record name |
| limit | int | Results per page (max 500) |
| offset | int | Pagination offset |
| order_by | string | Sort field |
| order | string | Sort direction (asc/desc) |

**Response:**

```json
{
  "zone": "example.com",
  "records": [
    {
      "id": "REC-12345",
      "name": "www",
      "type": "A",
      "value": "192.0.2.1",
      "ttl": 3600,
      "priority": null,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-15T10:30:00Z"
    },
    {
      "id": "REC-12346",
      "name": "@",
      "type": "MX",
      "value": "mail.example.com",
      "ttl": 3600,
      "priority": 10,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 15,
    "limit": 100,
    "offset": 0
  }
}
```

#### Add Record

```http
POST /zones/{zone}/records
```

**Request Body:**

```json
{
  "name": "mail",
  "type": "A",
  "value": "192.0.2.10",
  "ttl": 3600,
  "priority": null
}
```

**Response:**

```json
{
  "success": true,
  "record": {
    "id": "REC-12347",
    "name": "mail",
    "type": "A",
    "value": "192.0.2.10",
    "ttl": 3600,
    "priority": null,
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

#### Batch Add Records

```http
POST /zones/{zone}/records/batch
```

**Request Body:**

```json
{
  "records": [
    {
      "name": "www",
      "type": "A",
      "value": "192.0.2.1",
      "ttl": 3600
    },
    {
      "name": "mail",
      "type": "A",
      "value": "192.0.2.10",
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "MX",
      "value": "mail.example.com",
      "priority": 10,
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "TXT",
      "value": "v=spf1 mx ~all",
      "ttl": 3600
    }
  ],
  "continue_on_error": true
}
```

**Response:**

```json
{
  "success": true,
  "added": 4,
  "failed": 0,
  "records": [
    {"id": "REC-12347", "name": "www", "success": true},
    {"id": "REC-12348", "name": "mail", "success": true},
    {"id": "REC-12349", "name": "@ (MX)", "success": true},
    {"id": "REC-12350", "name": "@ (TXT)", "success": true}
  ]
}
```

#### Update Record

```http
PUT /zones/{zone}/records/{record_id}
```

**Request Body:**

```json
{
  "name": "www",
  "value": "192.0.2.5",
  "ttl": 7200,
  "priority": null
}
```

#### Delete Record

```http
DELETE /zones/{zone}/records/{record_id}
```

#### Batch Delete Records

```http
POST /zones/{zone}/records/delete-batch
```

**Request Body:**

```json
{
  "record_ids": [
    "REC-12345",
    "REC-12346",
    "REC-12347"
  ]
}
```

### Record Type Specific

#### Add A Record

```http
POST /zones/{zone}/records/a
```

**Request Body:**

```json
{
  "name": "www",
  "value": "192.0.2.1",
  "ttl": 3600
}
```

#### Add AAAA Record

```http
POST /zones/{zone}/records/aaaa
```

**Request Body:**

```json
{
  "name": "www",
  "value": "2001:db8::1",
  "ttl": 3600
}
```

#### Add CNAME Record

```http
POST /zones/{zone}/records/cname
```

**Request Body:**

```json
{
  "name": "blog",
  "value": "myblog.wordpress.com",
  "ttl": 3600
}
```

**Note:** CNAME records cannot coexist with other record types at the same name.

#### Add MX Record

```http
POST /zones/{zone}/records/mx
```

**Request Body:**

```json
{
  "name": "@",
  "value": "mail.example.com",
  "priority": 10,
  "ttl": 3600
}
```

#### Add TXT Record

```http
POST /zones/{zone}/records/txt
```

**Request Body:**

```json
{
  "name": "@",
  "value": "v=spf1 mx -all",
  "ttl": 3600
}
```

#### Add SPF Record

```http
POST /zones/{zone}/records/spf
```

**Request Body:**

```json
{
  "name": "@",
  "version": "v=spf1",
  "mechanisms": ["mx", "a", "include:_spf.google.com"],
  "qualifier": "-",
  "all": "fail",
  "ttl": 3600
}
```

**Response:**

```json
{
  "success": true,
  "record": {
    "id": "REC-12345",
    "name": "@",
    "type": "TXT",
    "value": "v=spf1 mx a include:_spf.google.com -all",
    "ttl": 3600
  }
}
```

#### Add DKIM Record

```http
POST /zones/{zone}/records/dkim
```

**Request Body:**

```json
{
  "selector": "google",
  "value": "p=your_dkim_public_key_here",
  "ttl": 3600
}
```

**Note:** Creates a TXT record with name `selector._domainkey.domain.com`

#### Add SRV Record

```http
POST /zones/{zone}/records/srv
```

**Request Body:**

```json
{
  "service": "_sip",
  "protocol": "_tcp",
  "name": "_sip._tcp",
  "value": "sip.example.com",
  "priority": 10,
  "weight": 5,
  "port": 5060,
  "ttl": 3600
}
```

#### Add CAA Record

```http
POST /zones/{zone}/records/caa
```

**Request Body:**

```json
{
  "name": "@",
  "flags": 0,
  "tag": "issue",
  "value": "letsencrypt.org",
  "ttl": 3600
}
```

### DNS Templates

#### List Templates

```http
GET /dns/templates
```

**Response:**

```json
{
  "templates": [
    {
      "id": "TMPL-12345",
      "name": "standard",
      "description": "Standard DNS setup with common records",
      "record_count": 5
    },
    {
      "id": "TMPL-12346",
      "name": "mail",
      "description": "Email-focused setup with MX and SPF",
      "record_count": 8
    }
  ]
}
```

#### Apply Template

```http
POST /zones/{zone}/apply-template
```

**Request Body:**

```json
{
  "template_id": "TMPL-12345",
  "options": {
    "replace_existing": false,
    "skip_conflicts": true
  }
}
```

### DNSSEC

#### Get DNSSEC Keys

```http
GET /zones/{zone}/dnssec
```

**Response:**

```json
{
  "zone": "example.com",
  "dnssec_enabled": true,
  "keys": [
    {
      "id": "KKEY-12345",
      "flags": 257,
      "protocol": 3,
      "algorithm": 13,
      "key_tag": 12345,
      "public_key": "base64_encoded_key...",
      "created_at": "2024-01-01T00:00:00Z",
      "status": "active"
    }
  ]
}
```

#### Add DS Record

```http
POST /zones/{zone}/dnssec/ds
```

**Request Body:**

```json
{
  "key_id": "KKEY-12345",
  "algorithm": 13,
  "digest_type": 2,
  "digest": "sha256_digest_value"
}
```

### Import/Export

#### Import Zone

```http
POST /zones/{zone}/import
```

**Request Body:**

```json
{
  "format": "bind",
  "data": "example.com. 3600 IN A 192.0.2.1\nwww 3600 IN CNAME example.com."
}
```

**Supported Formats:**
- `bind` - BIND zone file format
- `json` - JSON record array
- `csv` - CSV format

#### Export Zone

```http
GET /zones/{zone}/export
```

**Query Parameters:**

| Parameter | Description |
|-----------|-------------|
| format | Export format (bind/json/csv) |

**Response (BIND format):**

```
$ORIGIN example.com.
$TTL 3600
@  IN SOA ns1.dnsprovider.com. admin.example.com. (
    2024011501 ; Serial
    7200       ; Refresh
    3600       ; Retry
    1209600    ; Expire
    86400 )    ; Minimum TTL

@  IN NS ns1.dnsprovider.com.
@  IN NS ns2.dnsprovider.com.
@  IN A 192.0.2.1
www IN A 192.0.2.1
mail IN A 192.0.2.10
@  IN MX 10 mail.example.com.
@  IN TXT "v=spf1 mx -all"
```

### Health Checks

#### Check Record Health

```http
GET /zones/{zone}/records/{record_id}/health
```

**Response:**

```json
{
  "record_id": "REC-12345",
  "health_check": {
    "enabled": true,
    "interval": 60,
    "timeout": 10,
    "healthy_threshold": 3,
    "unhealthy_threshold": 3
  },
  "last_check": {
    "status": "healthy",
    "checked_at": "2024-01-15T10:30:00Z",
    "response_time_ms": 45
  },
  "history": [
    {
      "timestamp": "2024-01-15T10:30:00Z",
      "status": "healthy",
      "response_time_ms": 45
    }
  ]
}
```

## Webhooks

### DNS Event Webhooks

```json
{
  "events": [
    "DnsRecordCreated",
    "DnsRecordUpdated",
    "DnsRecordDeleted",
    "DnsZoneCreated",
    "DnsZoneDeleted",
    "DnsPropagationComplete"
  ],
  "endpoint": "https://your-app.com/webhooks/dns",
  "secret": "your_webhook_secret"
}
```

### Webhook Payload

```json
{
  "event": "DnsRecordCreated",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "zone": "example.com",
    "record": {
      "id": "REC-12345",
      "name": "www",
      "type": "A",
      "value": "192.0.2.1",
      "ttl": 3600
    }
  }
}
```

## Error Handling

### Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| DNS_ERROR_001 | 400 | Invalid zone name |
| DNS_ERROR_002 | 400 | Invalid record type |
| DNS_ERROR_003 | 400 | Invalid record value |
| DNS_ERROR_004 | 404 | Zone not found |
| DNS_ERROR_005 | 404 | Record not found |
| DNS_ERROR_006 | 409 | Record conflict |
| DNS_ERROR_007 | 429 | Rate limit exceeded |
| DNS_ERROR_008 | 500 | DNS provider error |
| DNS_ERROR_009 | 400 | CNAME conflict |
| DNS_ERROR_010 | 400 | Record limit exceeded |

## SDK Examples

### PHP SDK

```php
use WHMCS\Dns\ApiClient;

$client = new ApiClient([
    'api_key' => 'your_api_key'
]);

// Add records
$client->addRecord('example.com', [
    'name' => 'www',
    'type' => 'A',
    'value' => '192.0.2.1'
]);

// Batch add
$client->addRecords('example.com', [
    ['name' => 'www', 'type' => 'A', 'value' => '192.0.2.1'],
    ['name' => 'mail', 'type' => 'A', 'value' => '192.0.2.10'],
    ['name' => '@', 'type' => 'MX', 'value' => 'mail.example.com', 'priority' => 10]
]);
```

### Python SDK

```python
from whmcs_dns import DnsClient

client = DnsClient(api_key='your_api_key')

# Add SPF record
client.add_spf_record('example.com', 'v=spf1 mx -all')

# Export zone
zone_data = client.export_zone('example.com', format='bind')
```

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [Records Management](./whmcs-records-management.md)
- [DNS Templates](./whmcs-dns-templates.md)
