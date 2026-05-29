# WHMCS Custom API Endpoint From Scratch Workflow

## Description
Create custom API endpoints for WHMCS to extend functionality.

## Prerequisites
- WHMCS 7.0+
- PHP 8.1+
- Basic REST API knowledge

## Steps

### Step 1: Create API Handler Directory
```bash
mkdir -p /var/www/whmcs/includes/api/clicodes
```

### Step 2: Create API Endpoint
```php
<?php
/**
 * WHMCS Custom API - CLICodes API
 * Access via: /includes/api/clicodes/endpoint.php
 */

require_once __DIR__ . '/../../init.php';

// Set response type
header('Content-Type: application/json');

// Get request data
$method = $_SERVER['REQUEST_METHOD'];
$input = json_decode(file_get_contents('php://input'), true) ?? [];
$queryParams = $_GET;

// Validate authentication
$apiKey = $_SERVER['HTTP_X_API_KEY'] ?? $_GET['api_key'] ?? null;

if (!$apiKey) {
    http_response_code(401);
    echo json_encode(['error' => 'API key required']);
    exit;
}

// Verify API key
$validKey = validateApiKey($apiKey);
if (!$validKey) {
    http_response_code(403);
    echo json_encode(['error' => 'Invalid API key']);
    exit;
}

$userId = $validKey['user_id'];

// Route request
$action = $queryParams['action'] ?? $input['action'] ?? 'info';

try {
    $result = match ($action) {
        'get_invoices' => getInvoices($userId, $queryParams),
        'get_invoice' => getInvoice($userId, $input['invoice_id'] ?? 0),
        'get_services' => getServices($userId),
        'get_service' => getService($userId, $input['service_id'] ?? 0),
        'get_tickets' => getTickets($userId),
        'create_ticket' => createTicket($userId, $input),
        'update_profile' => updateProfile($userId, $input),
        default => throw new Exception('Unknown action: ' . $action),
    };
    
    echo json_encode([
        'success' => true,
        'data' => $result,
    ]);
    
} catch (Exception $e) {
    http_response_code(400);
    echo json_encode([
        'success' => false,
        'error' => $e->getMessage(),
    ]);
}

/**
 * Get user invoices
 */
function getInvoices($userId, $params)
{
    $status = $params['status'] ?? null;
    $limit = min((int)($params['limit'] ?? 50), 100);
    
    $query = Capsule::table('tblinvoices')
        ->where('userid', $userId)
        ->orderBy('date', 'desc')
        ->limit($limit);
    
    if ($status) {
        $query->where('status', $status);
    }
    
    $invoices = $query->get();
    
    return array_map(function($invoice) {
        return [
            'id' => $invoice->id,
            'invoice_num' => $invoice->invoicenum,
            'date' => $invoice->date,
            'duedate' => $invoice->duedate,
            'total' => $invoice->total,
            'status' => $invoice->status,
            'items' => getInvoiceItems($invoice->id),
        ];
    }, $invoices->toArray());
}

/**
 * Get single invoice
 */
function getInvoice($userId, $invoiceId)
{
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->where('userid', $userId)
        ->first();
    
    if (!$invoice) {
        throw new Exception('Invoice not found');
    }
    
    return [
        'id' => $invoice->id,
        'invoice_num' => $invoice->invoicenum,
        'date' => $invoice->date,
        'duedate' => $invoice->duedate,
        'total' => $invoice->total,
        'status' => $invoice->status,
        'items' => getInvoiceItems($invoice->id),
        'notes' => $invoice->notes,
    ];
}

/**
 * Get invoice line items
 */
function getInvoiceItems($invoiceId)
{
    return Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->get()
        ->map(function($item) {
            return [
                'description' => $item->description,
                'amount' => $item->amount,
                'taxed' => (bool)$item->taxed,
            ];
        })
        ->toArray();
}

/**
 * Get user services
 */
function getServices($userId)
{
    $services = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->orderBy('regdate', 'desc')
        ->get();
    
    return array_map(function($service) {
        return [
            'id' => $service->id,
            'domain' => $service->domain,
            'product' => getProductName($service->packageid),
            'regdate' => $service->regdate,
            'nextduedate' => $service->nextduedate,
            'status' => $service->domainstatus,
        ];
    }, $services->toArray());
}

/**
 * Get single service
 */
function getService($userId, $serviceId)
{
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->where('userid', $userId)
        ->first();
    
    if (!$service) {
        throw new Exception('Service not found');
    }
    
    return [
        'id' => $service->id,
        'domain' => $service->domain,
        'product' => getProductName($service->packageid),
        'regdate' => $service->regdate,
        'nextduedate' => $service->nextduedate,
        'status' => $service->domainstatus,
        'username' => $service->username,
        'subscription_id' => $service->subscriptionid,
    ];
}

/**
 * Get support tickets
 */
function getTickets($userId)
{
    $tickets = Capsule::table('tbltickets')
        ->where('userid', $userId)
        ->orderBy('created', 'desc')
        ->limit(50)
        ->get();
    
    return array_map(function($ticket) {
        return [
            'id' => $ticket->id,
            'tid' => $ticket->tid,
            'subject' => $ticket->subject,
            'status' => $ticket->status,
            'priority' => $ticket->priority,
            'created' => $ticket->created,
            'last_reply' => $ticket->lastreply,
        ];
    }, $tickets->toArray());
}

/**
 * Create support ticket
 */
function createTicket($userId, $input)
{
    $required = ['subject', 'message', 'priority'];
    foreach ($required as $field) {
        if (empty($input[$field])) {
            throw new Exception("Missing required field: $field");
        }
    }
    
    $result = localAPI('OpenTicket', [
        'clientid' => $userId,
        'subject' => $input['subject'],
        'message' => $input['message'],
        'priority' => $input['priority'],
        'deptid' => $input['department_id'] ?? 1,
    ]);
    
    if ($result['result'] !== 'success') {
        throw new Exception($result['message'] ?? 'Failed to create ticket');
    }
    
    return [
        'ticket_id' => $result['tid'],
        'ticket_number' => $result['ticketNumber'],
    ];
}

/**
 * Update user profile
 */
function updateProfile($userId, $input)
{
    $allowedFields = ['firstname', 'lastname', 'companyname', 'email', 'phonenumber'];
    $updateData = [];
    
    foreach ($allowedFields as $field) {
        if (isset($input[$field])) {
            $updateData[$field] = $input[$field];
        }
    }
    
    if (empty($updateData)) {
        throw new Exception('No valid fields to update');
    }
    
    Capsule::table('tblclients')
        ->where('id', $userId)
        ->update($updateData);
    
    return ['updated' => array_keys($updateData)];
}

/**
 * Validate API key
 */
function validateApiKey($key)
{
    $record = Capsule::table('mod_api_keys')
        ->where('api_key', hash('sha256', $key))
        ->where('active', 1)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->first();
    
    return $record ? ['user_id' => $record->user_id] : null;
}

/**
 * Get product name
 */
function getProductName($productId)
{
    return Capsule::table('tblproducts')
        ->where('id', $productId)
        ->value('name') ?? 'Unknown Product';
}
```

### Step 3: Create API Key Management
```php
<?php
/**
 * API Key Management Functions
 */

// Generate new API key
function generateApiKey($userId, $name, $expiresIn = '1 year')
{
    $key = bin2hex(random_bytes(32));
    $hashedKey = hash('sha256', $key);
    
    Capsule::table('mod_api_keys')->insert([
        'user_id' => $userId,
        'name' => $name,
        'api_key' => $hashedKey,
        'created_at' => date('Y-m-d H:i:s'),
        'expires_at' => date('Y-m-d H:i:s', strtotime($expiresIn)),
        'active' => 1,
    ]);
    
    return $key; // Return plain key - show only once
}

// Revoke API key
function revokeApiKey($keyId)
{
    Capsule::table('mod_api_keys')
        ->where('id', $keyId)
        ->update(['active' => 0]);
}

// Create database table
function installApiKeysTable()
{
    Capsule::schema()->create('mod_api_keys', function($table) {
        $table->increments('id');
        $table->integer('user_id');
        $table->string('name', 100);
        $table->string('api_key', 64)->unique();
        $table->timestamp('created_at')->useCurrent();
        $table->timestamp('expires_at')->nullable();
        $table->boolean('active')->default(1);
    });
}
```

### Step 4: Add API Routes
```php
<?php
/**
 * Alternative: Use custom route handler
 * Add to Configuration > System Settings > API Credentials
 */

// Example: WHMCS native API wrapper
$result = localAPI('GetClientsDetails', [
    'clientid' => $clientId,
    'stats' => true,
]);
```

### Step 5: Test API
```bash
# Test get_invoices
curl -X GET \
  "https://whmcs.example.com/includes/api/clicodes/endpoint.php?action=get_invoices&api_key=YOUR_API_KEY"

# Test create_ticket
curl -X POST \
  "https://whmcs.example.com/includes/api/clicodes/endpoint.php?action=create_ticket" \
  -H "Content-Type: application/json" \
  -d '{"api_key": "YOUR_API_KEY", "subject": "Test", "message": "Test ticket", "priority": "Medium"}'
```

## Security Considerations
- Always use HTTPS
- Implement rate limiting
- Log all API requests
- Use API key rotation
- Validate all input
- Use prepared statements

## Tags
- api
- rest
- development
- integration