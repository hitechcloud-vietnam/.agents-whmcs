# WHMCS API Documentation Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to creating and maintaining API documentation for WHMCS integrations, including endpoint documentation, parameter references, code examples, and best practices for developer experience.

## Prerequisites

- WHMCS installation with API access enabled
- OpenAPI/Swagger knowledge
- Markdown documentation tools
- API development environment

## Workflow Steps

### Step 1: Document API Endpoints

Create comprehensive API endpoint documentation:

```yaml
# api-docs/openapi.yaml

openapi: 3.0.3
info:
  title: WHMCS REST API
  description: |
    Complete API reference for WHMCS automation and integration.
    Supports client management, billing, service provisioning, and more.
  version: 1.0.0
  contact:
    email: api-support@yourcompany.com

servers:
  - url: https://your-whmcs.com/api/v1
    description: Production server
  - url: https://staging-whmcs.com/api/v1
    description: Staging server

paths:
  /clients:
    get:
      summary: List all clients
      description: Retrieve a paginated list of all clients
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 25
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
        - name: status
          in: query
          schema:
            type: string
            enum: [Active, Inactive, Closed]
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Client'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

    post:
      summary: Create new client
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ClientCreate'
      responses:
        '201':
          description: Client created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Client'

  /clients/{id}:
    get:
      summary: Get client by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Client'
        '404':
          description: Client not found

    put:
      summary: Update client
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ClientUpdate'
      responses:
        '200':
          description: Client updated

  /services:
    get:
      summary: List all services
      parameters:
        - name: client_id
          in: query
          schema:
            type: integer
        - name: status
          in: query
          schema:
            type: string
            enum: [Active, Suspended, Terminated, Pending]
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Service'

    post:
      summary: Create new service
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ServiceCreate'
      responses:
        '201':
          description: Service created

  /invoices:
    get:
      summary: List invoices
      parameters:
        - name: client_id
          in: query
          schema:
            type: integer
        - name: status
          in: query
          schema:
            type: string
            enum: [Paid, Unpaid, Overdue, Cancelled]
        - name: date_from
          in: query
          schema:
            type: string
            format: date
        - name: date_to
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Successful response

  /invoices/{id}/pay:
    post:
      summary: Pay an invoice
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - payment_method
              properties:
                payment_method:
                  type: string
                  example: banktransfer
                amount:
                  type: number
      responses:
        '200':
          description: Invoice paid successfully

components:
  schemas:
    Client:
      type: object
      properties:
        id:
          type: integer
          readOnly: true
        email:
          type: string
          format: email
        first_name:
          type: string
        last_name:
          type: string
        company:
          type: string
        phone:
          type: string
        address:
          type: object
          properties:
            address1:
              type: string
            address2:
              type: string
            city:
              type: string
            state:
              type: string
            postcode:
              type: string
            country:
              type: string
              example: US
        status:
          type: string
          enum: [Active, Inactive, Closed]
        created_at:
          type: string
          format: date-time
          readOnly: true

    Service:
      type: object
      properties:
        id:
          type: integer
          readOnly: true
        client_id:
          type: integer
        domain:
          type: string
        product:
          type: string
        status:
          type: string
        billing_cycle:
          type: string
          enum: [Monthly, Quarterly, Semi-Annual, Annual]
        amount:
          type: number
        next_due_date:
          type: string
          format: date

    Pagination:
      type: object
      properties:
        total:
          type: integer
        limit:
          type: integer
        offset:
          type: integer
        has_more:
          type: boolean

  securitySchemes:
    api_key:
      type: apiKey
      in: header
      name: X-API-Key
    bearer:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - api_key: []
  - bearer: []
```

### Step 2: Create Developer Reference Documentation

Build markdown-based developer documentation:

```markdown
# WHMCS API Developer Guide

## Authentication

### API Key Authentication

Include your API key in every request header:

```bash
curl -H "X-API-Key: your_api_key_here" \
     -H "Content-Type: application/json" \
     https://your-whmcs.com/api/v1/clients
```

### OAuth 2.0 Authentication

For OAuth 2.0 flows:

```php
// Step 1: Redirect to authorization
$authUrl = 'https://your-whmcs.com/oauth/authorize?' . http_build_query([
    'client_id'     => 'your_client_id',
    'redirect_uri'  => 'https://your-app.com/callback',
    'response_type' => 'code',
    'scope'         => 'read write',
]);

// Step 2: Exchange code for token
$token = exchangeCodeForToken($_GET['code']);

// Step 3: Use token in requests
$response = makeApiRequest('/api/v1/clients', [], $token);
```

## Client Management

### Create Client

```php
$clientData = [
    'email'      => 'client@example.com',
    'first_name' => 'John',
    'last_name'  => 'Doe',
    'company'    => 'Example Corp',
    'phone'      => '+1234567890',
    'address'    => [
        'address1' => '123 Main St',
        'city'    => 'New York',
        'state'   => 'NY',
        'postcode'=> '10001',
        'country' => 'US',
    ],
];

$response = $api->post('/clients', $clientData);

// Response:
// {
//   "success": true,
//   "data": {
//     "id": 12345,
//     "email": "client@example.com",
//     ...
//   }
// }
```

### Get Client

```php
$client = $api->get('/clients/12345');

// Response:
// {
//   "success": true,
//   "data": {
//     "id": 12345,
//     "email": "client@example.com",
//     "first_name": "John",
//     ...
//   }
// }
```

### Update Client

```php
$updateData = [
    'phone' => '+1987654321',
    'company' => 'New Company Name',
];

$response = $api->put('/clients/12345', $updateData);
```

## Service Management

### Create Service

```apache
POST /api/v1/services

{
  "client_id": 12345,
  "product_id": 1,
  "domain": "example.com",
  "billing_cycle": "Monthly",
  "custom_fields": {
    "cp_username": "exampleuser"
  }
}
```

### Suspend Service

```apache
POST /api/v1/services/67890/suspend

{
  "reason": "Payment overdue"
}
```

### Terminate Service

```apache
POST /api/v1/services/67890/terminate
```

## Invoice Operations

### List Invoices

```php
$filters = [
    'client_id'  => 12345,
    'status'     => 'Unpaid',
    'date_from'  => '2026-01-01',
    'date_to'    => '2026-12-31',
];

$invoices = $api->get('/invoices', $filters);
```

### Create Invoice

```php
$invoiceData = [
    'client_id' => 12345,
    'items' => [
        [
            'description' => 'Monthly Hosting',
            'amount'      => 9.99,
            'quantity'    => 1,
        ],
        [
            'description' => 'Setup Fee',
            'amount'      => 25.00,
            'quantity'    => 1,
        ],
    ],
    'due_date' => '2026-06-15',
];

$invoice = $api->post('/invoices', $invoiceData);
```

### Pay Invoice

```php
$payment = [
    'payment_method' => 'banktransfer',
    'amount'         => 34.99,
];

$response = $api->post('/invoices/54321/pay', $payment);
```

## Error Handling

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "CLIENT_NOT_FOUND",
    "message": "Client with ID 12345 not found",
    "details": {
      "id": 12345
    }
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|------------|-------------|
| `INVALID_API_KEY` | 401 | API key is invalid or expired |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 422 | Invalid request data |
| `RATE_LIMITED` | 429 | Too many requests |
| `SERVER_ERROR` | 500 | Internal server error |

## Rate Limiting

- **Default limit**: 100 requests per minute
- **Burst limit**: 20 requests per second
- Rate limit headers included in responses:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1623254400
```
```

### Step 3: Build Interactive Documentation Portal

Create a searchable documentation website:

```php
// modules/addons/api_docs/api_docs.php

function api_docs_config(): array
{
    return [
        'name'        => 'API Documentation',
        'description' => 'Interactive API documentation portal',
        'version'     => '1.0',
    ];
}

function api_docs_output(array $vars): void
{
    $action = $_REQUEST['action'] ?? 'overview';

    switch ($action) {
        case 'overview':
            include __DIR__ . '/views/overview.php';
            break;
        case 'endpoints':
            include __DIR__ . '/views/endpoints.php';
            break;
        case 'authentication':
            include __DIR__ . '/views/authentication.php';
            break;
        case 'sdk':
            include __DIR__ . '/views/sdk.php';
            break;
        case 'try-it':
            $this->renderApiTester();
            break;
    }
}

/**
 * API Tester functionality
 */
public function renderApiTester(): void
{
    echo '<div class="api-tester">';
    echo '<h2>API Tester</h2>';

    echo '<form id="apiTesterForm">';
    echo '<div class="form-group">';
    echo '<label for="endpoint">Endpoint:</label>';
    echo '<select name="endpoint" id="endpoint">';
    $this->renderEndpointOptions($endpoints);
    echo '</select>';
    echo '</div>';

    echo '<div class="form-group">';
    echo '<label for="method">Method:</label>';
    echo '<select name="method" id="method">';
    echo '<option value="GET">GET</option>';
    echo '<option value="POST">POST</option>';
    echo '<option value="PUT">PUT</option>';
    echo '<option value="DELETE">DELETE</option>';
    echo '</select>';
    echo '</div>';

    echo '<div class="form-group">';
    echo '<label for="payload">Request Body (JSON):</label>';
    echo '<textarea name="payload" id="payload" rows="10"></textarea>';
    echo '</div>';

    echo '<button type="submit">Send Request</button>';
    echo '</form>';

    echo '<div id="response" class="response-area">';
    echo '<h3>Response:</h3>';
    echo '<pre id="responseBody"></pre>';
    echo '</div>';
    echo '</div>';

    $this->renderTesterScript();
}

private function renderTesterScript(): void
{
?>
<script>
document.getElementById('apiTesterForm').addEventListener('submit', async function(e) {
    e.preventDefault();

    const endpoint = document.getElementById('endpoint').value;
    const method = document.getElementById('method').value;
    const payload = document.getElementById('payload').value;

    try {
        const response = await fetch('/api/v1' + endpoint, {
            method: method,
            headers: {
                'Content-Type': 'application/json',
                'X-API-Key': '<?= $_SESSION['adminapi_key'] ?>',
            },
            body: method !== 'GET' ? payload : undefined,
        });

        const data = await response.json();
        document.getElementById('responseBody').textContent = JSON.stringify(data, null, 2);
    } catch (error) {
        document.getElementById('responseBody').textContent = 'Error: ' + error.message;
    }
});
</script>
<?php
}
```

### Step 4: Generate SDK Code Examples

Create language-specific SDK examples:

```php
// modules/addons/api_docs/examples/php-sdk.php

/**
 * WHMCS PHP SDK Example
 */

class WhmcsApiClient
{
    private string $baseUrl;
    private string $apiKey;
    private int $timeout = 30;

    public function __construct(string $baseUrl, string $apiKey)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
    }

    public function getClient(int $clientId): array
    {
        return $this->request('GET', "/clients/{$clientId}");
    }

    public function createClient(array $data): array
    {
        return $this->request('POST', '/clients', $data);
    }

    public function updateClient(int $clientId, array $data): array
    {
        return $this->request('PUT', "/clients/{$clientId}", $data);
    }

    public function getInvoices(array $filters = []): array
    {
        $query = http_build_query($filters);
        return $this->request('GET', "/invoices?{$query}");
    }

    public function getServices(int $clientId): array
    {
        return $this->request('GET', "/services?client_id={$clientId}");
    }

    public function createService(array $data): array
    {
        return $this->request('POST', '/services', $data);
    }

    public function suspendService(int $serviceId, string $reason): array
    {
        return $this->request('POST', "/services/{$serviceId}/suspend", [
            'reason' => $reason,
        ]);
    }

    private function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->baseUrl . '/api/v1' . $endpoint;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-API-Key: ' . $this->apiKey,
            ],
        ]);

        if ($method === 'POST' || $method === 'PUT') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        if ($method === 'PUT') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
        }

        if ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new ApiException(
                $result['error']['message'] ?? 'API Error',
                $httpCode,
                $result['error']['code'] ?? 'UNKNOWN'
            );
        }

        return $result;
    }
}

// Usage Example
$api = new WhmcsApiClient('https://your-whmcs.com', 'your_api_key');

// Get client
$client = $api->getClient(12345);

// Create new client
$newClient = $api->createClient([
    'email'      => 'newclient@example.com',
    'first_name'  => 'Jane',
    'last_name'   => 'Smith',
    'company'     => 'New Company',
    'address'     => [
        'address1' => '456 Oak Ave',
        'city'    => 'Los Angeles',
        'state'   => 'CA',
        'postcode'=> '90001',
        'country' => 'US',
    ],
]);

// Get invoices
$invoices = $api->getInvoices([
    'client_id' => 12345,
    'status'   => 'Unpaid',
]);
```

---

## Best Practices

1. **Keep documentation up-to-date** - Update docs when APIs change
2. **Provide multiple code examples** - Show usage in common languages
3. **Include error handling patterns** - Help developers handle errors gracefully
4. **Document rate limits** - Clearly communicate usage limits
5. **Provide interactive testing** - Allow developers to try API calls
6. **Version your documentation** - Maintain version history
7. **Include authentication details** - Document all auth methods clearly
8. **Show real-world use cases** - Provide complete integration examples
9. **Document webhook events** - List all available webhook triggers
10. **Maintain changelog** - Track API changes over time

---

## Verification Checklist

- [ ] OpenAPI specification validates successfully
- [ ] All endpoints documented with examples
- [ ] Authentication methods clearly explained
- [ ] Error codes documented with solutions
- [ ] Interactive API tester functional
- [ ] Code examples work when copied
- [ ] SDK examples cover common use cases
- [ ] Documentation portal loads correctly
- [ ] Search functionality works
- [ ] Version history maintained
