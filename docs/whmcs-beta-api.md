# WHMCS Beta API Features

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-api-integration`, `whmcs-api-documentation`, `whmcs-rest-api-builder`

---

## Overview

WHMHCS continuously introduces new API features and endpoints. This guide covers beta API features, how to access them, and best practices for using experimental endpoints.

---

## Beta API Access

### Enabling Beta Features

```php
<?php
/**
 * Enable beta API access
 * Note: This requires admin access and appropriate permissions
 */

// Method 1: Via configuration
// Add to configuration.php
define('WHMCS_API_BETA_ENABLED', true);

// Method 2: Via admin settings
// Navigate to: Setup > General Settings > API
// Enable "Enable Beta API Features"

// Method 3: Per-request header
$headers = [
    'Authorization: Bearer ' . $apiKey,
    'X-API-Beta: true',
    'Accept: application/json',
];
```

### Beta API Endpoint Structure

```
https://your-whmcs.com/api/v2/beta/
├── clients/
│   ├── create/
│   ├── update/
│   └── delete/
├── orders/
│   ├── calculate/
│   └── preview/
├── services/
│   ├── provisioning/
│   └── management/
├── webhooks/
│   └── subscriptions/
└── graph/
    └── ql/           # GraphQL endpoint
```

---

## Beta Client Endpoints

### Create Client (Beta)

```php
<?php
/**
 * Create client - Beta version
 * Extended fields and validation
 */

$response = local_api('CreateClientBeta', [
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'address1' => '123 Main St',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US',
    'phonenumber' => '+1-555-123-4567',
    'password2' => 'securepassword123',  // Beta: Custom password
    'customfields' => [                   // Beta: Custom fields
        'field_1' => 'value',
    ],
    'metadata' => [                       // Beta: Additional metadata
        'source' => 'api',
        'referral_code' => 'REF123',
    ],
    'skip_validation' => false,           // Beta: Strict validation
]);

// Response includes beta-specific fields
$clientId = $response['clientid'];
$verificationStatus = $response['email_verified'];  // Beta field
$createdAt = $response['created_at'];                // Beta field
```

### Client Search (Beta)

```php
<?php
/**
 * Advanced client search with filters
 */

$response = local_api('SearchClientsBeta', [
    'filters' => [
        'status' => 'active',
        'has_outstanding_balance' => true,
        'created_after' => '2026-01-01',
        'has_service' => true,
        'tags' => ['premium', 'vip'],
    ],
    'sort' => [
        'field' => 'total_revenue',
        'order' => 'desc',
    ],
    'pagination' => [
        'page' => 1,
        'per_page' => 50,
    ],
    'include' => ['totals', 'services', 'invoices'],  // Related data
]);
```

---

## Beta Order Endpoints

### Order Preview

```php
<?php
/**
 * Preview order without creating it
 * Returns pricing breakdown and validation
 */

$response = local_api('PreviewOrderBeta', [
    'client_id' => 123,
    'products' => [
        [
            'product_id' => 1,
            'config_options' => [
                'billing_cycle' => 'monthly',
                'quantity' => 1,
            ],
        ],
    ],
    'promotion_code' => 'DISCOUNT20',
    'include_taxes' => true,
    'shipping_method' => 'standard',
]);

// Response structure
$preview = [
    'subtotal' => 99.00,
    'discount' => -19.80,
    'tax' => 7.14,
    'shipping' => 0.00,
    'total' => 86.34,
    'currency' => 'USD',
    'validation' => [
        'valid' => true,
        'warnings' => [],
        'errors' => [],
    ],
    'expires_at' => '2026-05-29T12:00:00Z',  // Price lock
];
```

### Order Calculation

```php
<?php
/**
 * Calculate order totals with various scenarios
 */

$response = local_api('CalculateOrderBeta', [
    'items' => [
        ['type' => 'product', 'id' => 1, 'qty' => 2],
        ['type' => 'addon', 'id' => 5, 'qty' => 1],
        ['type' => 'domain', 'id' => 'example.com', 'years' => 2],
    ],
    'client_id' => 123,
    'apply_credits' => true,
    'payment_method' => 'stripe',
    'coupon_code' => 'SAVE10',
]);
```

---

## Beta Service Endpoints

### Service Provisioning

```php
<?php
/**
 * Service provisioning with advanced options
 */

$response = local_api('ProvisionServiceBeta', [
    'client_id' => 123,
    'product_id' => 1,
    'domain' => 'newservice.example.com',
    'custom_fields' => [
        'cpu_cores' => 4,
        'ram_gb' => 8,
        'disk_gb' => 100,
    ],
    'config_options' => [
        'auto_setup' => true,
        'send_welcome_email' => true,
        'allow_terminal' => false,
    ],
    'scheduled_date' => '2026-06-01',  // Beta: Scheduled provisioning
    'notification_webhook' => 'https://myapp.com/webhook/provision',
]);

// Response
$service = [
    'service_id' => 456,
    'status' => 'provisioning',  // Not immediately active
    'estimated_ready' => '2026-05-29T12:30:00Z',
    'provisioning_steps' => [
        ['step' => 'create_account', 'status' => 'complete'],
        ['step' => 'configure_server', 'status' => 'complete'],
        ['step' => 'setup_dns', 'status' => 'pending'],
    ],
];
```

### Service Management

```php
<?php
/**
 * Advanced service management
 */

$response = local_api('ManageServiceBeta', [
    'service_id' => 456,
    'action' => 'resize',  // Upgrade/downgrade
    'parameters' => [
        'new_product_id' => 2,
        'prorate' => true,
        'immediate' => false,
        'reason' => 'Customer requested upgrade',
    ],
]);

/**
 * Service bulk operations
 */
$response = local_api('BulkServiceActionBeta', [
    'service_ids' => [456, 457, 458],
    'action' => 'suspend',
    'reason' => 'Non-payment',
    'notify_client' => true,
    'webhook' => 'https://myapp.com/webhook/bulk-status',
]);
```

---

## GraphQL API (Beta)

### GraphQL Endpoint

```
POST https://your-whmcs.com/api/graphql
Content-Type: application/json
Authorization: Bearer {api_key}
X-API-Beta: true
```

### GraphQL Examples

```php
<?php
/**
 * GraphQL query example
 */

$query = <<<'GRAPHQL'
query GetClientWithServices($clientId: ID!) {
    client(id: $clientId) {
        id
        email
        fullName
        status
        services {
            edges {
                node {
                    id
                    domain
                    status
                    product {
                        name
                        group {
                            name
                        }
                    }
                }
            }
        }
        invoices(status: "unpaid") {
            edges {
                node {
                    id
                    total
                    dueDate
                }
            }
        }
    }
}
GRAPHQL;

$variables = [
    'clientId' => '123',
];

$ch = curl_init('https://your-whmcs.com/api/graphql');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => json_encode([
        'query' => $query,
        'variables' => $variables,
    ]),
    CURLOPT_HTTPHEADER => [
        'Authorization: Bearer ' . $apiKey,
        'Content-Type: application/json',
        'X-API-Beta: true',
    ],
]);

$response = curl_exec($ch);
curl_close($ch);

$result = json_decode($response, true);
```

### GraphQL Mutations

```php
<?php
/**
 * GraphQL mutation example
 */

$mutation = <<<'GRAPHQL'
mutation CreateService($input: ServiceCreateInput!) {
    createService(input: $input) {
        service {
            id
            domain
            status
        }
        errors {
            field
            message
            code
        }
    }
}
GRAPHQL;

$variables = [
    'input' => [
        'clientId' => '123',
        'productId' => '1',
        'domain' => 'newservice.example.com',
        'customFields' => [
            'cpu_cores' => 4,
            'ram_gb' => 8,
        ],
    ],
];
```

---

## Webhook Subscriptions

```php
<?php
/**
 * Webhook subscription management (Beta)
 */

$response = local_api('CreateWebhookSubscription', [
    'url' => 'https://myapp.com/webhook/whmcs',
    'events' => [
        'client.created',
        'client.updated',
        'invoice.paid',
        'invoice.created',
        'service.created',
        'service.suspended',
        'service.terminated',
    ],
    'secret' => 'webhook_secret_123',
    'filters' => [
        'client_status' => ['active'],
    ],
    'active' => true,
]);

// Response
$subscription = [
    'id' => 'sub_abc123',
    'url' => 'https://myapp.com/webhook/whmcs',
    'events' => ['client.created', ...],
    'status' => 'active',
    'created_at' => '2026-05-29T00:00:00Z',
];

// Webhook payload example
$payload = [
    'id' => 'evt_xyz789',
    'type' => 'client.created',
    'timestamp' => '2026-05-29T12:00:00Z',
    'data' => [
        'client_id' => 123,
        'email' => 'new@example.com',
    ],
    'signature' => 'sha256=...',
];
```

---

## Usage Examples

### Integrating Beta Features

```php
<?php
/**
 * WHMCS Beta API Client
 */

class BetaApiClient
{
    private string $baseUrl;
    private string $apiKey;
    private bool $betaEnabled;

    public function __construct(string $baseUrl, string $apiKey, bool $betaEnabled = true)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
        $this->betaEnabled = $betaEnabled;
    }

    /**
     * Make API request
     */
    public function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->baseUrl . '/api/v2/' . ltrim($endpoint, '/');

        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
                'Accept: application/json',
                $this->betaEnabled ? 'X-API-Beta: true' : '',
            ],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception(
                $result['message'] ?? 'API error',
                $httpCode
            );
        }

        return $result;
    }

    /**
     * Create client with beta features
     */
    public function createClient(array $clientData): array
    {
        return $this->request('POST', 'beta/clients', $clientData);
    }

    /**
     * Preview order
     */
    public function previewOrder(array $orderData): array
    {
        return $this->request('POST', 'beta/orders/preview', $orderData);
    }

    /**
     * GraphQL query
     */
    public function graphql(string $query, array $variables = []): array
    {
        return $this->request('POST', 'graphql', [
            'query' => $query,
            'variables' => $variables,
        ]);
    }
}

// Usage
$client = new BetaApiClient(
    'https://whmcs.example.com',
    'your_api_key',
    true  // Enable beta features
);

$clientData = $client->createClient([
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
]);
```

---

## Best Practices for Beta APIs

1. **Check for stable alternatives** - Beta features may change
2. **Implement fallback** - Handle beta feature unavailability
3. **Version your usage** - Track which beta features you use
4. **Monitor for changes** - WHMCS may modify beta endpoints
5. **Test thoroughly** - Beta features may have edge cases
6. **Document your implementation** - Note beta feature dependencies

---

## Related Documentation

- [API Integration Patterns](api-integration-patterns.md)
- [API Gateway](api-gateway.md)
- [Webhook Events Reference](webhook-events-reference.md)
- [REST API Builder](../skills/whmcs-rest-api-builder.md)
