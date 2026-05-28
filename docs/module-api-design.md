# WHMCS Module API Design Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers RESTful API design for WHMCS modules, including endpoint design, request/response patterns, authentication, and documentation standards.

---

## API Structure

### Endpoint Conventions

```
https://yourwhmcs.com/modules/addons/your_addon/api/
├── v1/
│   ├── clients/              # Client operations
│   │   ├── GET    /          # List clients
│   │   ├── POST   /          # Create client
│   │   ├── GET    /{id}      # Get client details
│   │   ├── PUT    /{id}      # Update client
│   │   └── DELETE /{id}      # Delete client
│   ├── records/              # Module records
│   │   ├── GET    /          # List records
│   │   ├── POST   /          # Create record
│   │   └── GET    /{id}/stats # Get record stats
│   └── webhooks/             # Webhook management
│       ├── GET    /          # List webhooks
│       ├── POST   /          # Create webhook
│       └── DELETE /{id}      # Delete webhook
```

### API Handler

```php
<?php
/**
 * API Handler
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/../includes.php';

$requestMethod = $_SERVER['REQUEST_METHOD'];
$requestPath = $_SERVER['REQUEST_URI'];
$input = json_decode(file_get_contents('php://input'), true);

/**
 * Route the request
 */
function routeRequest($method, $path, $input)
{
    // Remove API prefix
    $path = preg_replace('#^/modules/addons/your_addon/api/v1/#', '', $path);
    $path = trim($path, '/');
    
    // Parse path segments
    $segments = explode('/', $path);
    
    // Route handling
    try {
        switch ($segments[0]) {
            case 'clients':
                return routeClients($method, $segments, $input);
                
            case 'records':
                return routeRecords($method, $segments, $input);
                
            default:
                throw new NotFoundException('Endpoint not found');
        }
    } catch (\Exception $e) {
        return handleError($e);
    }
}

/**
 * Route client requests
 */
function routeClients($method, $segments, $input)
{
    // GET /clients
    if ($method === 'GET' && count($segments) === 1) {
        return listClients();
    }
    
    // POST /clients
    if ($method === 'POST' && count($segments) === 1) {
        return createClient($input);
    }
    
    // GET /clients/{id}
    if ($method === 'GET' && count($segments) === 2 && is_numeric($segments[1])) {
        return getClient($segments[1]);
    }
    
    throw new NotFoundException('Endpoint not found');
}

function routeRecords($method, $segments, $input)
{
    // Handle records routes...
}
```

## Request Handling

### Input Validation

```php
/**
 * Validate API input
 */
function validateApiInput($input, $schema)
{
    $errors = [];
    
    foreach ($schema as $field => $rules) {
        $value = $input[$field] ?? null;
        
        // Required check
        if (in_array('required', $rules) && ($value === null || $value === '')) {
            $errors[$field] = 'This field is required';
            continue;
        }
        
        // Skip validation if not required and empty
        if ($value === null || $value === '') {
            continue;
        }
        
        // Type validation
        if (in_array('integer', $rules) && !filter_var($value, FILTER_VALIDATE_INT)) {
            $errors[$field] = 'Must be an integer';
        }
        
        if (in_array('email', $rules) && !filter_var($value, FILTER_VALIDATE_EMAIL)) {
            $errors[$field] = 'Must be a valid email';
        }
        
        if (in_array('string', $rules) && !is_string($value)) {
            $errors[$field] = 'Must be a string';
        }
        
        if (in_array('array', $rules) && !is_array($value)) {
            $errors[$field] = 'Must be an array';
        }
        
        // Length validation
        if (isset($rules['min_length']) && strlen($value) < $rules['min_length']) {
            $errors[$field] = "Minimum length is {$rules['min_length']}";
        }
        
        if (isset($rules['max_length']) && strlen($value) > $rules['max_length']) {
            $errors[$field] = "Maximum length is {$rules['max_length']}";
        }
    }
    
    return [
        'valid'   => empty($errors),
        'errors'  => $errors,
    ];
}

/**
 * Schema definitions
 */
function getClientSchema()
{
    return [
        'email' => ['required', 'email'],
        'firstname' => ['required', 'string', 'max_length' => 100],
        'lastname' => ['required', 'string', 'max_length' => 100],
        'companyname' => ['string', 'max_length' => 100],
    ];
}
```

## Response Format

### Standard Response

```php
/**
 * Send JSON response
 */
function sendResponse($data, $statusCode = 200)
{
    http_response_code($statusCode);
    header('Content-Type: application/json');
    echo json_encode($data);
    exit;
}

/**
 * Success response
 */
function successResponse($data, $message = null, $statusCode = 200)
{
    $response = [
        'success' => true,
        'data'    => $data,
    ];
    
    if ($message) {
        $response['message'] = $message;
    }
    
    sendResponse($response, $statusCode);
}

/**
 * Paginated response
 */
function paginatedResponse($data, $pagination)
{
    sendResponse([
        'success' => true,
        'data'    => $data,
        'pagination' => [
            'total'        => $pagination['total'],
            'page'         => $pagination['page'],
            'per_page'     => $pagination['per_page'],
            'total_pages'  => $pagination['total_pages'],
            'has_more'     => $pagination['page'] < $pagination['total_pages'],
        ],
    ]);
}

/**
 * Error response
 */
function errorResponse($message, $code = 'ERROR', $details = null, $statusCode = 400)
{
    $response = [
        'success' => false,
        'error' => [
            'code'    => $code,
            'message' => $message,
        ],
    ];
    
    if ($details) {
        $response['error']['details'] = $details;
    }
    
    sendResponse($response, $statusCode);
}
```

## Authentication

### API Key Authentication

```php
/**
 * Authenticate API request
 */
function authenticateApiRequest()
{
    // Get API key from header
    $headers = getallheaders();
    $apiKey = $headers['X-API-Key'] ?? $headers['Authorization'] ?? null;
    
    if (!$apiKey) {
        throw new UnauthorizedException('API key required');
    }
    
    // Remove Bearer prefix if present
    $apiKey = preg_replace('/^Bearer\s+/i', '', $apiKey);
    
    // Verify API key
    $keyData = WHMCS\Database\Capsule::table('mod_your_api_keys')
        ->where('api_key', $apiKey)
        ->where('active', 1)
        ->whereNull('deleted_at')
        ->first();
    
    if (!$keyData) {
        throw new UnauthorizedException('Invalid API key');
    }
    
    // Check expiration
    if ($keyData->expires_at && strtotime($keyData->expires_at) < time()) {
        throw new UnauthorizedException('API key expired');
    }
    
    // Update last used
    WHMCS\Database\Capsule::table('mod_your_api_keys')
        ->where('id', $keyData->id)
        ->update(['last_used_at' => date('Y-m-d H:i:s')]);
    
    return $keyData;
}

/**
 * Admin authentication
 */
function authenticateAdmin()
{
    if (!$_SESSION['adminid']) {
        throw new UnauthorizedException('Admin authentication required');
    }
    
    $admin = WHMCS\Database\Capsule::table('tbladmins')
        ->where('id', $_SESSION['adminid'])
        ->where('disabled', 0)
        ->first();
    
    if (!$admin) {
        throw new UnauthorizedException('Admin not found');
    }
    
    return $admin;
}
```

## Rate Limiting

```php
/**
 * Rate limiting
 */
function checkRateLimit($keyData)
{
    $limit = $keyData->rate_limit ?? 60; // Requests per minute
    $window = 60; // Seconds
    
    $now = time();
    $windowStart = date('Y-m-d H:i:s', $now - $window);
    
    // Count recent requests
    $recentRequests = WHMCS\Database\Capsule::table('mod_your_api_logs')
        ->where('api_key_id', $keyData->id)
        ->where('created_at', '>=', $windowStart)
        ->count();
    
    if ($recentRequests >= $limit) {
        throw new RateLimitException(
            'Rate limit exceeded. Limit: ' . $limit . '/minute'
        );
    }
    
    // Log this request
    WHMCS\Database\Capsule::table('mod_your_api_logs')
        ->insert([
            'api_key_id' => $keyData->id,
            'endpoint'   => $_SERVER['REQUEST_URI'],
            'method'     => $_SERVER['REQUEST_METHOD'],
            'ip_address' => $_SERVER['REMOTE_ADDR'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
}
```

## Webhook Endpoints

```php
/**
 * Register webhook handler
 */
function routeWebhooks($method, $segments, $input)
{
    // Ensure webhook signature verification
    verifyWebhookSignature();
    
    $action = $input['action'] ?? $input['event'] ?? null;
    
    if (!$action) {
        throw new BadRequestException('Webhook action/event required');
    }
    
    switch ($action) {
        case 'client.synced':
            return handleClientSynced($input['data']);
            
        case 'service.created':
            return handleServiceCreated($input['data']);
            
        case 'invoice.paid':
            return handleInvoicePaid($input['data']);
            
        default:
            // Acknowledge but don't process unknown events
            return successResponse(['received' => true]);
    }
}

/**
 * Process webhook
 */
function handleClientSynced($data)
{
    // Validate required fields
    $required = ['client_id', 'external_id'];
    foreach ($required as $field) {
        if (!isset($data[$field])) {
            throw new BadRequestException("Missing required field: {$field}");
        }
    }
    
    // Update internal records
    $clientId = WHMCS\Database\Capsule::table('mod_your_clients')
        ->where('external_id', $data['external_id'])
        ->value('client_id');
    
    if ($clientId) {
        WHMCS\Database\Capsule::table('mod_your_clients')
            ->where('external_id', $data['external_id'])
            ->update([
                'synced_at' => date('Y-m-d H:i:s'),
                'data'      => json_encode($data),
            ]);
    }
    
    return successResponse(['processed' => true]);
}
```

## Documentation Format

### OpenAPI/Swagger Documentation

```yaml
openapi: 3.0.0
info:
  title: Your Addon API
  version: 1.0.0
  description: API for Your Addon Module
  
servers:
  - url: https://yourwhmcs.com/modules/addons/your_addon/api/v1
    description: Production

paths:
  /clients:
    get:
      summary: List clients
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: per_page
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Client'
                      
    post:
      summary: Create client
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateClient'
      responses:
        '201':
          description: Created
          
components:
  schemas:
    Client:
      type: object
      properties:
        id:
          type: integer
        email:
          type: string
        firstname:
          type: string
        lastname:
          type: string
          
    CreateClient:
      type: object
      required:
        - email
        - firstname
        - lastname
      properties:
        email:
          type: string
        firstname:
          type: string
        lastname:
          type: string
          maxLength: 100
```

---

## Related Skills and Workflows

- `api-integration-patterns` - Integration patterns
- `api-endpoints-reference` - WHMCS API reference
- `module-security-standards` - API security
- `webhook-events-reference` - Webhook events
