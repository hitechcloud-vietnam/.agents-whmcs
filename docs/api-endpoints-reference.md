# WHMCS API Endpoints Reference

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `whmcs-api-integration`, `webhook-events-reference`

## Overview

WHMCS provides a comprehensive REST API for external integration. All API calls are made to the `/includes/api.php` endpoint or using the API v2 endpoint `/api/v2/`.

## Authentication

### API Authentication Methods

```php
// Method 1: Using credentials directly
$postData = [
    'action' => 'GetClients',
    'username' => 'admin_user',
    'password' => 'hashed_password', // Use md5 or create access key
    'responsetype' => 'json',
];

// Method 2: Using API Credentials (recommended)
$postData = [
    'action' => 'GetClients',
    'api_identifier' => 'your_api_identifier',
    'api_secret' => 'your_api_secret',
    'responsetype' => 'json',
];
```

## Client Management

### GetClients
Retrieve a list of clients with optional filters.

```php
$postData = [
    'action' => 'GetClients',
    'limitstart' => 0,
    'limitnum' => 100,
    'sorting' => 'client_id',
    'search' => 'john@example.com',
    'status' => 'Active',
];

// Response
{
    "result": "success",
    "totalresults": 150,
    "startnumber": 0,
    "numreturned": 100,
    "clients": {
        "client": [
            {
                "id": "1",
                "email": "john@example.com",
                "firstname": "John",
                "lastname": "Doe",
                "companyname": "Acme Corp",
                "status": "Active"
            }
        ]
    }
}
```

### GetClientDetails
Retrieve detailed information about a specific client.

```php
$postData = [
    'action' => 'GetClientDetails',
    'client_id' => 123,
    'stats' => true,
    'customfields' => true,
    'groups' => true,
    'currency' => true,
];
```

### AddClient
Create a new client account.

```php
$postData = [
    'action' => 'AddClient',
    'firstname' => 'Jane',
    'lastname' => 'Smith',
    'email' => 'jane@example.com',
    'companyname' => 'Tech Solutions',
    'password2' => 'securePassword123', // Plain text, will be hashed
    'country' => 'US',
    'state' => 'CA',
    'city' => 'San Francisco',
    'address1' => '123 Main St',
    'phonenumber' => '+1-555-123-4567',
    'currency' => 1,
    'clientgroupid' => 1,
    'customfields' => base64_encode(serialize(['field_id' => 'value'])),
    'skipValidation' => false,
];
```

### UpdateClient
Update existing client information.

```php
$postData = [
    'action' => 'UpdateClient',
    'client_id' => 123,
    'firstname' => 'Jane',
    'lastname' => 'Smith Updated',
    'email' => 'newemail@example.com',
    'notes' => 'VIP customer since 2020',
];
```

### CloseClient
Close a client account.

```php
$postData = [
    'action' => 'CloseClient',
    'client_id' => 123,
    'close_related_services' => true,
];
```

## Orders and Products

### GetOrders
Retrieve order history.

```php
$postData = [
    'action' => 'GetOrders',
    'limitstart' => 0,
    'limitnum' => 50,
    'id' => 123, // Optional specific order ID
    'user_id' => 456, // Optional client ID
    'status' => 'Active',
    'orderby' => 'id',
    'sorting' => 'DESC',
];
```

### AddOrder
Create a new order.

```php
$postData = [
    'action' => 'AddOrder',
    'client_id' => 123,
    'pid' => [1, 2], // Product IDs (can be array)
    'billingcycle' => ['monthly', 'annually'],
    'domaintype' => 'register',
    'domain' => ['example.com', 'example.org'],
    'regperiod' => [1, 2],
    'paymentmethod' => 'paypal',
    'promocode' => 'DISCOUNT20',
    'addons' => [5, 8], // Optional addon IDs
    'configoptions' => [
        'configid_1' => 'value',
        'configid_2' => 'value',
    ],
    'customfields' => base64_encode(serialize(['field_id' => 'value'])),
    'noemail' => true, // Skip confirmation email
];
```

### AcceptOrder
Accept a pending order.

```php
$postData = [
    'action' => 'AcceptOrder',
    'orderid' => 123,
    'sendemail' => true,
    'auto_setup' => true,
];
```

### GetProducts
Retrieve product list.

```php
$postData = [
    'action' => 'GetProducts',
    'pid' => 1, // Optional specific product
    'gid' => 1, // Optional product group
    'module' => 'cpanel',
    'stats' => true,
];
```

## Billing and Invoices

### GetInvoices
Retrieve invoices.

```php
$postData = [
    'action' => 'GetInvoices',
    'user_id' => 123,
    'status' => 'Paid',
    'limitstart' => 0,
    'limitnum' => 25,
    'date' => [
        'from' => '2026-01-01',
        'to' => '2026-05-28',
    ],
];
```

### CreateInvoice
Generate a new invoice.

```php
$postData = [
    'action' => 'CreateInvoice',
    'user_id' => 123,
    'items' => [
        [
            'description' => 'Web Hosting - Monthly',
            'amount' => 9.99,
            'taxed' => true,
        ],
        [
            'description' => 'Domain Renewal',
            'amount' => 14.99,
            'taxed' => false,
        ],
    ],
    'sendinvoice' => true,
    'due_date' => '2026-06-15',
    'notes' => 'Thank you for your business!',
];
```

### AddInvoicePayment
Record a payment against an invoice.

```php
$postData = [
    'action' => 'AddInvoicePayment',
    'invoice_id' => 123,
    'transaction_id' => 'TXN-123456',
    'amount' => 24.98,
    'gateway' => 'paypal',
    'date' => '2026-05-28',
    'no_email' => false,
];
```

### ApplyCredit
Apply account credit to an invoice.

```php
$postData = [
    'action' => 'ApplyCredit',
    'client_id' => 123,
    'invoice_id' => 456,
    'amount' => 50.00,
];
```

## Domain Management

### GetDomains
Retrieve domain list.

```php
$postData = [
    'action' => 'GetDomains',
    'client_id' => 123,
    'limitstart' => 0,
    'limitnum' => 100,
    'domain' => 'example.com',
    'status' => 'Active',
];
```

### RegisterDomain
Register a new domain.

```php
$postData = [
    'action' => 'RegisterDomain',
    'client_id' => 123,
    'domain' => 'newdomain.com',
    'regperiod' => 2,
    'dnsmanagement' => true,
    'emailforwarding' => false,
    'idprotection' => true,
    'registrar' => 'enom',
    'paymentmethod' => 'paypal',
];
```

### TransferDomain
Initiate a domain transfer.

```php
$postData = [
    'action' => 'TransferDomain',
    'client_id' => 123,
    'domain' => 'transfer.com',
    'transfersecret' => 'auth_code_here',
    'registrar' => 'enom',
    'paymentmethod' => 'paypal',
];
```

### RenewDomain
Renew a domain registration.

```php
$postData = [
    'action' => 'RenewDomain',
    'client_id' => 123,
    'domain' => 'example.com',
    'regperiod' => 1,
    'paymentmethod' => 'paypal',
];
```

## Support Tickets

### GetTickets
Retrieve support tickets.

```php
$postData = [
    'action' => 'GetTickets',
    'clientid' => 123,
    'ticketid' => 456,
    'deptid' => 1,
    'status' => 'Open',
    'subject' => 'billing issue',
];
```

### OpenTicket
Create a new support ticket.

```php
$postData = [
    'action' => 'OpenTicket',
    'clientid' => 123,
    'deptid' => 2,
    'subject' => 'Technical Support Request',
    'message' => 'I am experiencing issues with my hosting...',
    'priority' => 'Medium', // Low, Medium, High, Urgent
    'attachments' => [
        ['name' => 'screenshot.png', 'data' => base64_encode($fileContent)],
    ],
];
```

### AddTicketReply
Reply to an existing ticket.

```php
$postData = [
    'action' => 'AddTicketReply',
    'ticketid' => 456,
    'clientid' => 123,
    'message' => 'Thank you for the update. Please see attached logs.',
    'attachments' => [],
];
```

## Module Commands

### ModuleCreate
Execute provisioning create command.

```php
$postData = [
    'action' => 'ModuleCreate',
    'serviceid' => 789,
    'username' => 'new_user_account',
];
```

### ModuleSuspend
Suspend a service.

```php
$postData = [
    'action' => 'ModuleSuspend',
    'serviceid' => 789,
    'suspend_reason' => 'Non-payment',
];
```

### ModuleTerminate
Terminate a service.

```php
$postData = [
    'action' => 'ModuleTerminate',
    'serviceid' => 789,
];
```

### ModuleChangePassword
Change service password.

```php
$postData = [
    'action' => 'ModuleChangePassword',
    'serviceid' => 789,
    'newpassword' => 'NewSecurePass123!',
];
```

## API Response Handling

### Success Response

```json
{
    "result": "success",
    "totalresults": 100,
    "message": "Operation completed successfully"
}
```

### Error Response

```json
{
    "result": "error",
    "message": "Client ID not found",
    "errorcode": 1001
}
```

## Rate Limiting

Implement rate limiting in your integration:

```php
class WHMCSAPIClient {
    private $maxRequestsPerMinute = 60;
    private $requestLog = [];

    public function makeRequest(array $postData): array {
        $now = time();

        // Clean old entries
        $this->requestLog = array_filter(
            $this->requestLog,
            fn($timestamp) => $timestamp > $now - 60
        );

        if (count($this->requestLog) >= $this->maxRequestsPerMinute) {
            throw new Exception('Rate limit exceeded. Wait before retrying.');
        }

        $this->requestLog[] = $now;

        // Make API request
        return $this->executeRequest($postData);
    }
}
```

## Best Practices

1. **Use API Credentials**: Never hardcode admin credentials
2. **Implement Retry Logic**: Handle temporary failures gracefully
3. **Log All Requests**: Maintain audit trail for debugging
4. **Use Pagination**: Always respect limitstart/limitnum for large datasets
5. **Validate Input**: Sanitize all user-supplied data before API calls
6. **Handle Webhooks**: Subscribe to relevant webhook events for real-time updates

## Related Documentation

- [Webhook Events Reference](webhook-events-reference.md)
- [Hooks Reference](hooks-reference.md)
- [Email Template Variables](email-template-variables.md)
