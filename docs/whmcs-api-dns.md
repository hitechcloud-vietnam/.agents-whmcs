# WHMCS DNS API Endpoints Documentation

## Overview

Complete API reference for WHMCS DNS management operations.

## Authentication

All DNS API endpoints require authentication:

```http
Authorization: Bearer your_api_key
Content-Type: application/json
```

## Base URL

```
https://api.whmcs.com/v1/dns
```

## Endpoints Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /zones | List all zones |
| POST | /zones | Create zone |
| GET | /zones/{zone} | Get zone details |
| PUT | /zones/{zone} | Update zone |
| DELETE | /zones/{zone} | Delete zone |
| GET | /zones/{zone}/records | List records |
| POST | /zones/{zone}/records | Add record |
| PUT | /zones/{zone}/records/{id} | Update record |
| DELETE | /zones/{zone}/records/{id} | Delete record |
| POST | /zones/{zone}/apply-template | Apply template |

## Zone Management

### List Zones

```http
GET /zones
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| limit | int | Results per page (default 100, max 500) |
| offset | int | Pagination offset |
| status | string | Filter by status (active, suspended, pending) |
| tld | string | Filter by TLD |

**Response:**

```json
{
  "zones": [
    {
      "zone": "example.com",
      "zone_id": "ZONE-001",
      "status": "active",
      "record_count": 12,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-15T10:00:00Z"
    }
  ],
  "pagination": {
    "total": 150,
    "limit": 100,
    "offset": 0,
    "has_more": true
  }
}
```

### Create Zone

```http
POST /zones
```

**Request Body:**

```json
{
  "domain": "newdomain.com",
  "template": "standard-web",
  "variables": {
    "server_ip": "192.0.2.1"
  },
  "nameservers": [
    "ns1.your-dns.com",
    "ns2.your-dns.com"
  ],
  "ttl": 3600
}
```

**Response:**

```json
{
  "success": true,
  "zone": {
    "zone": "newdomain.com",
    "zone_id": "ZONE-002",
    "status": "active",
    "record_count": 0,
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

### Get Zone Details

```http
GET /zones/{zone}
```

**Response:**

```json
{
  "zone": "example.com",
  "zone_id": "ZONE-001",
  "status": "active",
  "ttl": 3600,
  "soa": {
    "primary_ns": "ns1.registrar.com",
    "admin_email": "admin@example.com",
    "serial": "2024011501",
    "refresh": 7200,
    "retry": 3600,
    "expire": 1209600,
    "minimum_ttl": 86400
  },
  "nameservers": [
    "ns1.registrar.com",
    "ns2.registrar.com"
  ],
  "dnssec": {
    "enabled": true,
    "keys_count": 2
  },
  "record_count": 12,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-15T10:00:00Z"
}
```

### Update Zone

```http
PUT /zones/{zone}
```

**Request Body:**

```json
{
  "ttl": 7200,
  "soa": {
    "refresh": 14400,
    "retry": 7200
  }
}
```

### Delete Zone

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

## Record Management

### List Records

```http
GET /zones/{zone}/records
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| type | string | Filter by record type |
| name | string | Filter by record name |
| limit | int | Results per page |
| offset | int | Pagination offset |

**Response:**

```json
{
  "zone": "example.com",
  "records": [
    {
      "id": "REC-001",
      "name": "@",
      "type": "A",
      "value": "192.0.2.1",
      "ttl": 3600,
      "priority": null,
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "REC-002",
      "name": "www",
      "type": "CNAME",
      "value": "@",
      "ttl": 3600,
      "priority": null,
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "REC-003",
      "name": "@",
      "type": "MX",
      "value": "mail.example.com",
      "ttl": 3600,
      "priority": 10,
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 12,
    "limit": 100,
    "offset": 0,
    "has_more": false
  }
}
```

### Add Record

```http
POST /zones/{zone}/records
```

**Request Body:**

```json
{
  "name": "api",
  "type": "A",
  "value": "192.0.2.100",
  "ttl": 3600
}
```

**Response:**

```json
{
  "success": true,
  "record": {
    "id": "REC-004",
    "name": "api",
    "type": "A",
    "value": "192.0.2.100",
    "ttl": 3600,
    "priority": null,
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

### Add Multiple Records (Batch)

```http
POST /zones/{zone}/records/batch
```

**Request Body:**

```json
{
  "records": [
    {"name": "www", "type": "A", "value": "192.0.2.1", "ttl": 3600},
    {"name": "mail", "type": "A", "value": "192.0.2.10", "ttl": 3600},
    {"name": "@", "type": "MX", "value": "mail.example.com", "priority": 10, "ttl": 3600}
  ]
}
```

### Update Record

```http
PUT /zones/{zone}/records/{record_id}
```

**Request Body:**

```json
{
  "name": "www",
  "value": "192.0.2.5",
  "ttl": 7200
}
```

### Delete Record

```http
DELETE /zones/{zone}/records/{record_id}
```

**Response:**

```json
{
  "success": true,
  "record_id": "REC-004",
  "deleted_at": "2024-01-15T10:30:00Z"
}
```

### Delete Multiple Records

```http
POST /zones/{zone}/records/delete-batch
```

**Request Body:**

```json
{
  "record_ids": ["REC-001", "REC-002", "REC-003"]
}
```

## Record Type Endpoints

### Add A Record

```http
POST /zones/{zone}/records/a
```

```json
{
  "name": "www",
  "value": "192.0.2.1",
  "ttl": 3600
}
```

### Add AAAA Record

```http
POST /zones/{zone}/records/aaaa
```

```json
{
  "name": "www",
  "value": "2001:db8::1",
  "ttl": 3600
}
```

### Add CNAME Record

```http
POST /zones/{zone}/records/cname
```

```json
{
  "name": "blog",
  "value": "myblog.wordpress.com",
  "ttl": 3600
}
```

### Add MX Record

```http
POST /zones/{zone}/records/mx
```

```json
{
  "name": "@",
  "value": "mail.example.com",
  "priority": 10,
  "ttl": 3600
}
```

### Add TXT Record

```http
POST /zones/{zone}/records/txt
```

```json
{
  "name": "@",
  "value": "v=spf1 mx -all",
  "ttl": 3600
}
```

### Add SPF Record

```http
POST /zones/{zone}/records/spf
```

```json
{
  "name": "@",
  "mechanisms": ["mx", "a"],
  "qualifier": "-",
  "all": "all"
}
```

**Response:**

```json
{
  "success": true,
  "record": {
    "id": "REC-005",
    "name": "@",
    "type": "TXT",
    "value": "v=spf1 mx a -all",
    "ttl": 3600
  }
}
```

### Add DKIM Record

```http
POST /zones/{zone}/records/dkim
```

```json
{
  "selector": "google",
  "value": "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8..."
}
```

### Add SRV Record

```http
POST /zones/{zone}/records/srv
```

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

### Add CAA Record

```http
POST /zones/{zone}/records/caa
```

```json
{
  "name": "@",
  "flags": 0,
  "tag": "issue",
  "value": "letsencrypt.org",
  "ttl": 3600
}
```

## DNSSEC

### Get DNSSEC Info

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
      "id": "KEY-001",
      "flags": 257,
      "protocol": 3,
      "algorithm": 13,
      "key_tag": 12345,
      "public_key": "base64_key...",
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "ds_records": [
    "example.com. 3600 IN DS 12345 13 2 AABBCC..."
  ]
}
```

### Enable DNSSEC

```http
POST /zones/{zone}/dnssec/enable
```

**Response:**

```json
{
  "success": true,
  "zone": "example.com",
  "dnssec_enabled": true,
  "ds_records": [
    "example.com. 3600 IN DS 12345 13 2 ..."
  ]
}
```

### Disable DNSSEC

```http
POST /zones/{zone}/dnssec/disable
```

## Templates

### Apply Template

```http
POST /zones/{zone}/apply-template
```

**Request Body:**

```json
{
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

### List Templates

```http
GET /dns/templates
```

## Import/Export

### Import Zone

```http
POST /zones/{zone}/import
```

**Request Body:**

```json
{
  "format": "bind",
  "data": "$ORIGIN example.com.\n@ IN A 192.0.2.1\nwww IN A 192.0.2.1"
}
```

### Export Zone

```http
GET /zones/{zone}/export
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| format | string | Export format: bind, json, csv |

## Search

### Search Records

```http
GET /dns/search
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| query | string | Search term |
| zone | string | Filter by zone |
| type | string | Filter by record type |
| limit | int | Results limit |

**Response:**

```json
{
  "results": [
    {
      "zone": "example.com",
      "record": {
        "id": "REC-001",
        "name": "www",
        "type": "A",
        "value": "192.0.2.1"
      }
    }
  ],
  "total": 1
}
```

## Webhooks

### Supported Events

| Event | Description |
|-------|-------------|
| `DnsRecordCreated` | Record added |
| `DnsRecordUpdated` | Record modified |
| `DnsRecordDeleted` | Record removed |
| `DnsZoneCreated` | Zone created |
| `DnsZoneDeleted` | Zone deleted |
| `DnsZoneUpdated` | Zone modified |
| `DnssecEnabled` | DNSSEC enabled |
| `DnssecDisabled` | DNSSEC disabled |

### Webhook Payload

```json
{
  "event": "DnsRecordCreated",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "zone": "example.com",
    "record": {
      "id": "REC-004",
      "name": "api",
      "type": "A",
      "value": "192.0.2.100"
    }
  }
}
```

## Error Responses

### Error Format

```json
{
  "success": false,
  "error": {
    "code": "DNS_ERROR_001",
    "message": "Zone not found",
    "details": "The zone example.com does not exist"
  }
}
```

### Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| DNS_ERROR_001 | 404 | Zone not found |
| DNS_ERROR_002 | 400 | Invalid zone name |
| DNS_ERROR_003 | 404 | Record not found |
| DNS_ERROR_004 | 400 | Invalid record type |
| DNS_ERROR_005 | 400 | Invalid record value |
| DNS_ERROR_006 | 409 | Record conflict |
| DNS_ERROR_007 | 429 | Rate limit exceeded |
| DNS_ERROR_008 | 500 | DNS provider error |
| DNS_ERROR_009 | 400 | CNAME conflict |
| DNS_ERROR_010 | 400 | Record limit exceeded |

## Rate Limits

| Plan | Requests/Minute | Requests/Day |
|------|-----------------|--------------|
| Standard | 60 | 10,000 |
| Professional | 300 | 50,000 |
| Enterprise | 1,000 | 200,000 |

## See Also

- [DNS Management API](./whmcs-dns-management-api.md)
- [Zone Editor](./whmcs-zone-editor.md)
- [Records Management](./whmcs-records-management.md)
