# WHMCS API Development Workflow

## Purpose
Integrate with WHMCS using the REST API

## Prerequisites
- WHMCS installed
- API credentials
- HTTP client (cURL/PHP/Python)

## Step 1: Enable WHMCS API

Navigate to: Setup > System Settings > API Credentials

Click "Create New API Credential"

## Step 2: Create API Access Key

```
Identifier: my_api_key
Description: Production API Access
Allowed IPs: [your server IPs]
Permissions: [select required]
```

## Step 3: Understand API Endpoint

Base URL:
```
https://yourdomain.com/whmcs/api/v2/
```

## Step 4: Authentication

### Using Access Key
```bash
curl -X POST "https://yourdomain.com/whmcs/api/v2/orders" \
  -H "Authorization: Bearer YOUR_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": 1,
    "pid": [1],
    "billingcycle": "monthly",
    "domain": "example.com"
  }'
```

### Using Admin Credentials
```bash
curl -X POST "https://yourdomain.com/whmcs/api/v2/orders" \
  -H "Authorization: Basic BASE64(username:password)" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

## Step 5: PHP API Client Example

```php
<?php
class WHMCSApiClient
{
    private $baseUrl;
    private $apiKey;
    
    public function __construct($baseUrl, $apiKey)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
    }
    
    public function request($endpoint, $data = [], $method = 'POST')
    {
        $ch = curl_init();
        
        $url = $this->baseUrl . '/api/v2/' . $endpoint;
        
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        if ($method !== 'GET' && !empty($data)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'code' => $httpCode,
            'data' => json_decode($response, true),
        ];
    }
    
    public function getClients($params = [])
    {
        return $this->request('clients', $params);
    }
    
    public function getClient($clientId)
    {
        return $this->request('clients/' . $clientId);
    }
    
    public function createOrder($data)
    {
        return $this->request('orders', $data);
    }
    
    public function getInvoices($params = [])
    {
        return $this->request('invoices', $params);
    }
}
```

## Step 6: Common API Endpoints

### Clients
```
GET    /clients                  - List clients
POST   /clients                 - Create client
GET    /clients/{id}            - Get client details
PUT    /clients/{id}            - Update client
DELETE /clients/{id}            - Delete client
```

### Orders
```
GET    /orders                  - List orders
POST   /orders                  - Create order
GET    /orders/{id}            - Get order details
POST   /orders/{id}/pending     - Accept pending order
POST   /orders/{id}/cancel      - Cancel order
```

### Invoices
```
GET    /invoices                - List invoices
POST   /invoices                - Create invoice
GET    /invoices/{id}          - Get invoice details
POST   /invoices/{id}/paid     - Mark as paid
POST   /invoices/{id}/refund   - Refund invoice
```

### Products
```
GET    /products                - List products
POST   /products                - Create product
GET    /products/{id}          - Get product details
PUT    /products/{id}          - Update product
DELETE /products/{id}          - Delete product
```

### Tickets
```
GET    /support/tickets         - List tickets
POST   /support/tickets         - Create ticket
GET    /support/tickets/{id}   - Get ticket details
POST   /support/tickets/{id}/reply - Reply to ticket
```

## Step 7: API Response Format

### Success Response
```json
{
    "result": "success",
    "data": {
        "id": 1,
        "name": "John Doe",
        "email": "john@example.com"
    },
    "meta": {
        "page": 1,
        "total": 100
    }
}
```

### Error Response
```json
{
    "result": "error",
    "message": "Invalid client ID",
    "error": {
        "code": "CLIENT_NOT_FOUND",
        "field": "client_id"
    }
}
```

## Step 8: Rate Limiting

WHMCS API rate limits:
- 60 requests per minute per IP
- 600 requests per hour per API key

Handle rate limits:
```php
if ($response['code'] === 429) {
    $retryAfter = $response['headers']['Retry-After'] ?? 60;
    sleep($retryAfter);
    return $this->request($endpoint, $data, $method);
}
```

## Step 9: Webhook Configuration

Navigate to: Setup > System Settings > Webhooks

Create webhook:
```
URL: https://yourdomain.com/webhook.php
Events: [select events]
Secret: [webhook secret]
```

## Step 10: API Best Practices

1. **Use HTTPS always**
2. **Store credentials securely**
3. **Implement retry logic**
4. **Log all API calls**
5. **Handle rate limits**
6. **Validate input data**
7. **Use appropriate timeouts**

## API Development Checklist

- [ ] API enabled
- [ ] Credentials created
- [ ] Client library built
- [ ] Authentication configured
- [ ] Endpoints tested
- [ ] Rate limiting handled
- [ ] Error handling implemented
- [ ] Webhooks configured
