# WHMCS LocalAPI Usage Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for using WHMCS LocalAPI for inter-module communication, automation tasks, and programmatic access to WHMCS functionality.

## When to Use

- Calling WHMCS API from within WHMCS (cron jobs, hooks, addons)
- Module-to-module communication
- Automated client management operations
- Integration with external systems within WHMCS context

## LocalAPI Configuration

### 1. Authentication Setup

```php
<?php
// Initialize LocalAPI access
require_once __DIR__ . '/init.php';

// Method 1: Using Admin credentials (for admin operations)
$apiUsername = 'admin_username';
$apiAccessKey = 'your_admin_api_key'; // Found in WHMCS Admin > API Credentials

// Method 2: Using Admin ID (for hook/cron context)
$adminUser = \WHMCS\User\Admin::find(1);
```

### 2. Direct LocalAPI Class Usage

```php
<?php
use WHMCS\Authentication\TwoFactor\BackupCode;

class LocalApiClient {
    private string $baseUrl;
    private string $apiIdentifier;
    private string $apiSecret;

    public function __construct(string $apiIdentifier, string $apiSecret) {
        $this->baseUrl = \App::getSystemURL();
        $this->apiIdentifier = $apiIdentifier;
        $this->apiSecret = $apiSecret;
    }

    public function call(string $action, array $params = []): array {
        $postData = array_merge($params, [
            'action' => $action,
            'username' => $this->apiIdentifier,
            'password' => $this->apiSecret,
            'responsetype' => 'json',
        ]);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . 'includes/api.php',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($result['result'] === 'error') {
            throw new \Exception($result['message'] ?? 'API Error');
        }

        return $result;
    }
}
```

### 3. Using WHMCS Native LocalAPI Class

```php
<?php
// WHMCS 8.x+ LocalAPI call pattern
$command = 'GetClients';
$values = [
    'search' => 'test@example.com',
    'limitstart' => 0,
    'limitnum' => 10,
];

$api = new \WHMCS\Api\Application\Services\LocalApi();
$result = $api->call($command, $values, $adminUser);
```

## Common API Commands

### Client Management

```php
<?php
// Get Client Details
$command = 'GetClientsDetails';
$values = [
    'clientid' => 123,
    'stats' => true,
];
$result = localApiCall($command, $values);

// Add New Client
$command = 'AddClient';
$values = [
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'password2' => 'securePassword123!',
    'country' => 'US',
    'state' => 'CA',
    'city' => 'San Francisco',
    'address1' => '123 Main St',
    'phonenumber' => '+1-555-123-4567',
    'password' => 'remoteAuthPassword', // For remote authentication
];
$result = localApiCall($command, $values);

// Update Client
$command = 'UpdateClient';
$values = [
    'clientid' => 123,
    'firstname' => 'John',
    'lastname' => 'Smith',
    'email' => 'newemail@example.com',
];
$result = localApiCall($command, $values);

// Close/Delete Client
$command = 'CloseClient';
$values = [
    'clientid' => 123,
];
$result = localApiCall($command, $values);
```

### Invoice Operations

```php
<?php
// Create Invoice
$command = 'CreateInvoice';
$values = [
    'userid' => 123,
    'date' => date('Y-m-d'),
    'duedate' => date('Y-m-d', strtotime('+14 days')),
    'items' => [
        'item' => [
            [
                'description' => 'Service Renewal',
                'amount' => 9.99,
            ],
            [
                'description' => 'Additional Services',
                'amount' => 5.00,
            ],
        ],
    ],
    'sendinvoice' => true,
];
$result = localApiCall($command, $values);

// Add Invoice Payment
$command = 'AddInvoicePayment';
$values = [
    'invoiceid' => 456,
    'transid' => 'TXN-' . time(),
    'amount' => 14.99,
    'gateway' => 'paypal',
    'date' => date('Y-m-d H:i:s'),
];
$result = localApiCall($command, $values);

// Get Invoice
$command = 'GetInvoice';
$values = [
    'invoiceid' => 456,
];
$result = localApiCall($command, $values);
```

### Service Management

```php
<?php
// Create Module Account
$command = 'ModuleCreate';
$values = [
    'accountid' => 789, // serviceid
    'serviceid' => 789,
    'username' => 'newuser',
    'password' => encrypt('password123'),
];
$result = localApiCall($command, $values);

// Suspend Service
$command = 'ModuleSuspend';
$values = [
    'serviceid' => 789,
    'suspendreason' => 'Payment overdue',
];
$result = localApiCall($command, $values);

// Unsuspend Service
$command = 'ModuleUnsuspend';
$values = [
    'serviceid' => 789,
];
$result = localApiCall($command, $values);

// Terminate Service
$command = 'ModuleTerminate';
$values = [
    'serviceid' => 789,
    'terminatereason' => 'Customer request',
];
$result = localApiCall($command, $values);

// Change Service Package
$command = 'ModuleChangePackage';
$values = [
    'serviceid' => 789,
    'newproductid' => 15,
];
$result = localApiCall($command, $values);
```

### Domain Operations

```php
<?php
// Register Domain
$command = 'RegisterDomain';
$values = [
    'domainid' => 101,
    'registrar' => 'enom',
];
$result = localApiCall($command, $values);

// Transfer Domain
$command = 'TransferDomain';
$values = [
    'domainid' => 101,
    'transfersecret' => 'AUTHCODE123',
];
$result = localApiCall($command, $values);

// Renew Domain
$command = 'RenewDomain';
$values = [
    'domainid' => 101,
    'period' => 1,
];
$result = localApiCall($command, $values);

// Update Nameservers
$command = 'UpdateNameserver';
$values = [
    'domainid' => 101,
    'ns1' => 'ns1.newprovider.com',
    'ns2' => 'ns2.newprovider.com',
];
$result = localApiCall($command, $values);
```

### Order Processing

```php
<?php
// Accept Order
$command = 'AcceptOrder';
$values = [
    'orderid' => 1001,
    'sendregistrar' => true,
    'sendemail' => true,
];
$result = localApiCall($command, $values);

// Place Order
$command = 'PlaceOrder';
$values = [
    'clientid' => 123,
    'pid' => [1, 2], // Product IDs
    'billingcycle' => ['monthly', 'annually'],
    'configoptions' => [
        1 => ['optionid' => 5, 'qty' => 2], // For configurable options
    ],
    'domain' => ['example.com'],
    'paymentmethod' => 'paypal',
    'promocode' => 'DISCOUNT10',
];
$result = localApiCall($command, $values);

// Cancel Order
$command = 'CancelOrder';
$values = [
    'orderid' => 1001,
    'cancelimmediate' => true,
];
$result = localApiCall($command, $values);
```

### Ticket Operations

```php
<?php
// Open Ticket
$command = 'OpenTicket';
$values = [
    'clientid' => 123,
    'deptid' => 1,
    'subject' => 'Support Request',
    'message' => 'I need help with...',
    'priority' => 'Medium', // Low, Medium, High, Urgent
];
$result = localApiCall($command, $values);

// Reply to Ticket
$command = 'AddTicketReply';
$values = [
    'ticketid' => 500,
    'message' => 'Thank you for your response...',
    'clientid' => 123,
];
$result = localApiCall($command, $values);

// Get Ticket
$command = 'GetTicket';
$values = [
    'ticketid' => 500,
];
$result = localApiCall($command, $values);
```

## Hook Integration with LocalAPI

```php
<?php
// hooks.php
use WHMCS\Database\Capsule;

add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];

    // Fetch service details via LocalAPI
    $command = 'GetClientProduct';
    $values = [
        'serviceid' => $serviceId,
    ];
    $service = localApiCall($command, $values);

    // Create external system account
    $externalSystem = new ExternalSystemClient();
    $externalSystem->createAccount([
        'username' => $service['username'],
        'email' => $service['email'],
        'product' => $service['productname'],
    ]);

    // Log activity
    logActivity("External account created for service: {$serviceId}");
});

add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];

    // Get invoice details
    $invoice = localApiCall('GetInvoice', ['invoiceid' => $invoiceId]);

    // Update external billing system
    syncToBillingSystem($invoiceId, $invoice['total']);
});
```

## Cron Job Integration

```php
<?php
// includes/hooks/cron_tasks.php

add_hook('DailyCronJob', 1, function($vars) {
    // Sync all client data to external CRM
    $page = 0;
    $perPage = 100;

    do {
        $clients = localApiCall('GetClients', [
            'limitstart' => $page * $perPage,
            'limitnum' => $perPage,
            'sorting' => 'ASC',
        ]);

        foreach ($clients['clients']['client'] as $client) {
            syncToCRM($client);
        }

        $page++;
    } while (count($clients['clients']['client']) === $perPage);

    logActivity("CRM sync completed: {$page} pages processed");
});

add_hook('DailyCronJob', 2, function($vars) {
    // Check for overdue invoices and suspend services
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('duedate', '<', date('Y-m-d'))
        ->where('duedate', '>=', date('Y-m-d', strtotime('-7 days')))
        ->get();

    foreach ($overdueInvoices as $invoice) {
        // Find related overdue services
        $services = localApiCall('GetInvoice', [
            'invoiceid' => $invoice->id,
        ]);

        foreach ($services['items']['item'] as $item) {
            if (!empty($item['type']) && $item['type'] === 'HostingAccount') {
                // Suspend service (example - customize as needed)
                logActivity("Service {$item['relid']} pending suspension for unpaid invoice #{$invoice->id}");
            }
        }
    }
});
```

## Error Handling

```php
<?php
function localApiCall(string $command, array $values, $adminUser = null): array {
    try {
        // Method 1: Direct API call
        $postData = array_merge($values, [
            'action' => $command,
            'username' => $adminUser ? $adminUser->username : 'admin',
            'password' => $adminUser ? $adminUser->password : '', // hashed
            'responsetype' => 'json',
        ]);

        // Method 2: Using LocalApiService (WHMCS 8.x)
        $localApi = new \WHMCS\Api\Application\Services\LocalApi();
        $result = $localApi->call($command, $values, $adminUser);

        if ($result['result'] === 'error') {
            throw new \Exception($result['message'] ?? 'Unknown API error');
        }

        return $result;

    } catch (\Exception $e) {
        logActivity("LocalAPI Error - Command: {$command} - " . $e->getMessage());
        throw $e;
    }
}

// Usage with error handling
try {
    $result = localApiCall('UpdateClient', [
        'clientid' => 123,
        'email' => 'new@example.com',
    ]);
} catch (\Exception $e) {
    // Log and handle gracefully
    logActivity("Failed to update client: " . $e->getMessage());
    // Maybe notify admin or rollback changes
}
```

## Security Best Practices

```php
<?php
// 1. Use API credentials with limited permissions
// Create dedicated API users in WHMCS Admin

// 2. Never expose API keys in client-facing code
// Store in configuration or environment variables

// 3. Log all API calls
function localApiCall(string $command, array $values): array {
    logActivity("LocalAPI Call: {$command} - " . json_encode($values));

    // ... execute call ...

    logActivity("LocalAPI Response: {$command} - Result: " . ($result['result'] ?? 'unknown'));
    return $result;
}

// 4. Validate all inputs before API calls
$clientId = (int) ($_POST['clientid'] ?? 0);
if ($clientId <= 0) {
    throw new \InvalidArgumentException('Invalid client ID');
}

// 5. Rate limiting for automated tasks
$lastCall = Capsule::table('mod_api_calls')
    ->where('command', $command)
    ->orderBy('created_at', 'DESC')
    ->first();

if ($lastCall && (time() - strtotime($lastCall->created_at)) < 1) {
    usleep(500000); // Wait 0.5 seconds
}
```

## Checklist

- [ ] API credentials properly configured
- [ ] Admin user has appropriate permissions
- [ ] All API calls logged for debugging
- [ ] Error handling implemented
- [ ] Rate limiting for bulk operations
- [ ] Security: no sensitive data in logs
- [ ] Timeout values set appropriately
- [ ] Response validation on all calls

---

**Related Skills:**
- whmcs-api-integration
- whmcs-hooks-development
- whmcs-cron-automation
- whmcs-logging
- whmcs-error-handling

**Reference:**
- WHMCS LocalAPI Documentation: https://developers.whmcs.com/api-local/