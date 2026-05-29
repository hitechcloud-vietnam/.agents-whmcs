# WHMCS Webhook Development Workflow

## Purpose
Set up webhooks for real-time event notifications

## Prerequisites
- WHMCS installed
- External service to receive webhooks
- Admin access

## Step 1: Understand Webhooks

Webhooks send HTTP POST requests to your endpoint when events occur in WHMCS.

## Step 2: Create Webhook Handler

```php
<?php
// webhook_handler.php

// Verify webhook signature
$secret = 'your_webhook_secret';
$signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'];

$payload = file_get_contents('php://input');
$expectedSignature = hash_hmac('sha256', $payload, $secret);

if (!hash_equals($expectedSignature, $signature)) {
    http_response_code(401);
    die('Invalid signature');
}

// Parse payload
$data = json_decode($payload, true);

$eventType = $data['event_type'];
$eventData = $data['data'];

// Handle events
switch ($eventType) {
    case 'ClientCreated':
        handleNewClient($eventData);
        break;
    
    case 'InvoicePaid':
        handleInvoicePaid($eventData);
        break;
    
    case 'OrderCreated':
        handleNewOrder($eventData);
        break;
    
    case 'ServiceCreated':
        handleNewService($eventData);
        break;
    
    case 'TicketOpened':
        handleNewTicket($eventData);
        break;
    
    default:
        logEvent($eventType, $eventData);
}

http_response_code(200);

function handleNewClient($data)
{
    $clientId = $data['id'];
    $email = $data['email'];
    
    // Sync to external CRM
    syncToCRM($clientId);
}

function handleInvoicePaid($data)
{
    $invoiceId = $data['id'];
    $amount = $data['total'];
    
    // Update accounting system
    updateAccounting($invoiceId, $amount);
}
```

## Step 3: Configure WHMCS Webhooks

Navigate to: Setup > System Settings > Webhooks

Click "Create New Webhook"

### Webhook Settings
```
Name: External Integration
URL: https://yourdomain.com/webhook_handler.php
Secret: [generate secret]
```

### Enable Events
Select events to trigger webhook:
- [x] Client Created
- [x] Client Updated
- [x] Invoice Created
- [x] Invoice Paid
- [x] Invoice Cancelled
- [x] Order Created
- [x] Order Fulfilled
- [x] Service Created
- [x] Service Suspended
- [x] Service Terminated
- [x] Ticket Opened
- [x] Ticket Replied

### Filters
```
Client Groups: All
Products: All
Status: All
```

## Step 4: Configure Outgoing Webhook (Push)

Navigate to: Setup > System Settings > Webhooks

### Create Outgoing Webhook
```
Direction: Outgoing
URL: https://external-service.com/webhook
Events: [select]
Authentication: Bearer Token
Token: [your-token]
```

## Step 5: Test Webhook

### Send Test Event
Navigate to: Utilities > Webhooks

Click "Send Test Event" for any webhook.

### Test Payload Example
```json
{
    "event_type": "InvoicePaid",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": {
        "id": 123,
        "invoice_number": "INV-1001",
        "client_id": 456,
        "subtotal": 100.00,
        "total": 100.00,
        "status": "Paid",
        "paid_date": "2024-01-15"
    }
}
```

## Step 6: Implement Retry Logic

```php
<?php
// webhook_handler.php - with retry

$maxRetries = 3;
$retryDelay = 60; // seconds

function handleWebhook($data, $attempt = 1)
{
    try {
        $result = processWebhook($data);
        return $result;
    } catch (Exception $e) {
        if ($attempt < $maxRetries) {
            sleep($retryDelay);
            return handleWebhook($data, $attempt + 1);
        }
        throw $e;
    }
}
```

## Step 7: Log Webhook Events

```php
function logWebhook($eventType, $data, $response)
{
    \WHMCS\Database\Capsule::table('tblwebhook_logs')->insert([
        'event_type' => $eventType,
        'payload' => json_encode($data),
        'response' => json_encode($response),
        'created_at' => Carbon::now(),
    ]);
}
```

## Webhook Checklist

- [ ] Webhook handler created
- [ ] WHMCS webhooks configured
- [ ] Outgoing webhook set up
- [ ] Webhook tested
- [ ] Retry logic implemented
- [ ] Logging configured
