# WHMCS Sync API Documentation

## Overview

The WHMCS Sync API provides programmatic access to domain synchronization operations. This API enables integration with external systems, custom sync workflows, and real-time synchronization.

## Authentication

### API Key Authentication

Include your API key in the request header:

```http
Authorization: Bearer your_api_key_here
Content-Type: application/json
```

### Rate Limiting

| Plan | Requests/Minute | Requests/Day |
|------|-----------------|--------------|
| Standard | 60 | 10,000 |
| Professional | 300 | 50,000 |
| Enterprise | 1,000 | 200,000 |

## Base URL

```
Production: https://api.whmcs.com/v1/sync
Sandbox:    https://sandbox-api.whmcs.com/v1/sync
```

## Endpoints

### Domain Sync

#### Sync Single Domain

```http
POST /domains/{domain}/sync
```

**Request Body:**

```json
{
  "force": false,
  "include_history": true
}
```

**Response:**

```json
{
  "success": true,
  "domain": "example.com",
  "synced_at": "2024-01-15T10:30:00Z",
  "data": {
    "status": "active",
    "expiry_date": "2025-01-15",
    "nameservers": [
      "ns1.registrar.com",
      "ns2.registrar.com"
    ],
    "registrant": {
      "name": "John Doe",
      "email": "jdoe@example.com"
    },
    "privacy": {
      "enabled": true,
      "expires_at": "2025-01-15"
    },
    "transfer_status": "none"
  },
  "changes": {
    "status": null,
    "nameservers": null,
    "expiry_date": null
  }
}
```

#### Bulk Sync Domains

```http
POST /domains/sync
```

**Request Body:**

```json
{
  "domains": [
    "example.com",
    "example.net",
    "example.org"
  ],
  "options": {
    "parallel": true,
    "max_concurrent": 5,
    "continue_on_error": true
  }
}
```

**Response:**

```json
{
  "success": true,
  "batch_id": "BATCH-12345",
  "total": 3,
  "completed": 3,
  "failed": 0,
  "results": [
    {
      "domain": "example.com",
      "success": true,
      "synced_at": "2024-01-15T10:30:00Z"
    },
    {
      "domain": "example.net",
      "success": true,
      "synced_at": "2024-01-15T10:30:01Z"
    },
    {
      "domain": "example.org",
      "success": true,
      "synced_at": "2024-01-15T10:30:02Z"
    }
  ]
}
```

#### Sync Domains by Filter

```http
POST /domains/sync/filter
```

**Request Body:**

```json
{
  "filters": {
    "registrar": "enom",
    "status": ["active", "expired"],
    "expiry_before": "2024-02-01",
    "expiry_after": "2024-01-01"
  },
  "options": {
    "limit": 100,
    "offset": 0
  }
}
```

### Status Sync

#### Get Domain Status

```http
GET /domains/{domain}/status
```

**Response:**

```json
{
  "domain": "example.com",
  "status": "active",
  "status_code": 100,
  "status_description": "OK",
  "registrar_status": "OK",
  "registry_status": "active",
  "client_status": "active",
  "synced_at": "2024-01-15T10:30:00Z"
}
```

#### Sync Domain Status

```http
POST /domains/{domain}/status/sync
```

**Request Body:**

```json
{
  "check_registrar": true,
  "check_registry": true,
  "check_whois": true
}
```

### Expiration Sync

#### Get Expiration Info

```http
GET /domains/{domain}/expiration
```

**Response:**

```json
{
  "domain": "example.com",
  "expiry_date": "2025-01-15",
  "days_until_expiry": 365,
  "auto_renew": true,
  "renewal_status": "pending",
  "grace_period": {
    "active": true,
    "ends_at": "2025-02-14",
    "days_remaining": 30,
    "redemption_fee": 50.00
  },
  "pending_delete": {
    "active": false,
    "starts_at": null
  }
}
```

#### Sync Expiration Dates

```http
POST /domains/expiration/sync
```

**Request Body:**

```json
{
  "sync_type": "all",
  "update_whmcs": true,
  "send_notifications": true
}
```

### Nameserver Sync

#### Get Nameservers

```http
GET /domains/{domain}/nameservers
```

**Response:**

```json
{
  "domain": "example.com",
  "nameservers": [
    {
      "host": "ns1.registrar.com",
      "ip": "192.0.2.1",
      "glue": true
    },
    {
      "host": "ns2.registrar.com",
      "ip": "192.0.2.2",
      "glue": true
    }
  ],
  "varnish_ns": null,
  "synced_at": "2024-01-15T10:30:00Z"
}
```

#### Update Nameservers

```http
PUT /domains/{domain}/nameservers
```

**Request Body:**

```json
{
  "nameservers": [
    "ns1.newregistrar.com",
    "ns2.newregistrar.com"
  ],
  "update_registry": true,
  "propagate": true
}
```

### Contact Sync

#### Get Registrant Info

```http
GET /domains/{domain}/contacts
```

**Response:**

```json
{
  "domain": "example.com",
  "registrant": {
    "name": "John Doe",
    "organization": "Example Inc.",
    "email": "jdoe@example.com",
    "address": {
      "street1": "123 Main St",
      "street2": "Suite 100",
      "city": "Anytown",
      "state": "CA",
      "postal_code": "12345",
      "country": "US"
    },
    "phone": "+1.5551234567",
    "fax": "+1.5551234568"
  },
  "admin": { ... },
  "tech": { ... },
  "billing": { ... },
  "privacy_enabled": true
}
```

#### Update Registrant

```http
PUT /domains/{domain}/contacts
```

**Request Body:**

```json
{
  "registrant": {
    "name": "Jane Doe",
    "email": "jane@example.com"
  },
  "update_registry": true
}
```

### Sync Queue

#### Get Sync Queue

```http
GET /sync/queue
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| status | string | Filter by status (pending, processing, completed, failed) |
| type | string | Filter by sync type |
| limit | int | Number of results (max 100) |
| offset | int | Pagination offset |

**Response:**

```json
{
  "queue": [
    {
      "id": "SYNC-12345",
      "domain": "example.com",
      "type": "status_check",
      "priority": 2,
      "status": "pending",
      "scheduled_at": "2024-01-15T10:30:00Z",
      "attempts": 0
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

#### Add to Sync Queue

```http
POST /sync/queue
```

**Request Body:**

```json
{
  "domain": "example.com",
  "type": "full_sync",
  "priority": 2,
  "scheduled_at": "2024-01-15T12:00:00Z",
  "options": {
    "include_whois": true,
    "include_dns": true
  }
}
```

#### Process Queue

```http
POST /sync/queue/process
```

**Request Body:**

```json
{
  "batch_size": 50,
  "workers": 4,
  "continue_on_error": true
}
```

### Sync History

#### Get Sync History

```http
GET /domains/{domain}/sync/history
```

**Response:**

```json
{
  "domain": "example.com",
  "history": [
    {
      "id": "HIST-12345",
      "synced_at": "2024-01-15T10:30:00Z",
      "sync_type": "scheduled",
      "changes": {
        "nameservers": {
          "from": ["ns1.old.com", "ns2.old.com"],
          "to": ["ns1.new.com", "ns2.new.com"]
        }
      },
      "status": "completed",
      "duration_ms": 245
    }
  ],
  "pagination": {
    "total": 50,
    "limit": 20,
    "offset": 0
  }
}
```

## Webhooks

### Sync Event Webhooks

Configure webhooks for sync events:

```json
{
  "events": [
    "DomainSyncCompleted",
    "DomainSyncFailed",
    "DomainStatusChanged",
    "DomainExpired",
    "DomainTransferredIn",
    "DomainTransferredOut"
  ],
  "endpoint": "https://your-app.com/webhooks/sync",
  "secret": "your_webhook_secret"
}
```

### Webhook Payload

```json
{
  "event": "DomainStatusChanged",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "domain": "example.com",
    "domain_id": 12345,
    "client_id": 67890,
    "previous_status": "active",
    "new_status": "expired",
    "synced_at": "2024-01-15T10:30:00Z"
  }
}
```

## Error Handling

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "SYNC_ERROR_001",
    "message": "Domain not found",
    "details": "The domain example.com does not exist in WHMCS"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| SYNC_ERROR_001 | 404 | Domain not found |
| SYNC_ERROR_002 | 400 | Invalid domain format |
| SYNC_ERROR_003 | 429 | Rate limit exceeded |
| SYNC_ERROR_004 | 500 | Registrar API error |
| SYNC_ERROR_005 | 409 | Sync already in progress |
| SYNC_ERROR_006 | 403 | Insufficient permissions |
| SYNC_ERROR_007 | 503 | Registrar unavailable |

## SDK Examples

### PHP SDK

```php
use WHMCS\Sync\ApiClient;

$client = new ApiClient([
    'api_key' => 'your_api_key',
    'base_url' => 'https://api.whmcs.com/v1/sync'
]);

// Sync single domain
$result = $client->syncDomain('example.com');

// Bulk sync
$result = $client->syncDomains([
    'example.com',
    'example.net',
    'example.org'
]);

// Check queue status
$queue = $client->getQueue(['status' => 'pending']);
```

### Python SDK

```python
from whmcs_sync import SyncClient

client = SyncClient(api_key='your_api_key')

# Sync single domain
result = client.sync_domain('example.com')

# Bulk sync with options
result = client.sync_domains(
    domains=['example.com', 'example.net'],
    parallel=True,
    max_concurrent=5
)

# Get sync history
history = client.get_sync_history('example.com')
```

## See Also

- [Sync Daemon Configuration](./whmcs-sync-daemon.md)
- [DNS Management API](./whmcs-dns-management-api.md)
- [Registrar Commands](./whmcs-registrar-commands.md)
