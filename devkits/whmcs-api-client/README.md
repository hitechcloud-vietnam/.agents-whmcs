# WHMCS API Client DevKit

A comprehensive WHMCS API client supporting both Local API and REST API integration.

## Features

- Support for WHMCS Local API and REST API
- Credential management with secure storage
- Built-in request logging
- Comprehensive client/service/invoice/domain/ticket operations
- Timeout handling and error management

## Installation

1. Copy the `api-client.php` file to your WHMCS module directory:
   ```
   modules/addons/whmcs_api_client/api-client.php
   ```

2. Copy the `lib/` directory contents to:
   ```
   modules/addons/whmcs_api_client/lib/
   ```

3. Activate the module in WHMCS Admin > Addon Modules

## Configuration

### Local API Setup

1. In WHMCS Admin, go to **Setup > Staff Management > API Credentials**
2. Create an API credential with Admin access
3. Copy the Username and Password (or Access Key)

### REST API Setup

1. Create a REST API integration in your WHMCS installation
2. Obtain the API Key and Secret

## Usage

### Basic Local API Call

```php
use WHMCS\Database\Capsule;

// Include the client
require_once __DIR__ . '/lib/LocalApiClient.php';

$client = new \WhmcsApi\LocalApiClient(
    'https://your-whmcs.com/includes/api.php',
    'admin_username',
    'admin_password_or_access_key'
);

// Get clients
$clients = $client->getClients(['limitnum' => 20]);

// Add a client
$result = $client->addClient([
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'password2' => 'secure_password',
    'country' => 'US',
]);
```

### Basic REST API Call

```php
require_once __DIR__ . '/lib/RestApiClient.php';

$client = new \WhmcsApi\RestApiClient(
    'https://your-whmcs.com/api/v1',
    'your-api-key'
);

// Get clients
$clients = $client->getClients(['limit' => 20]);

// Create client
$client->createClient([
    'first_name' => 'Jane',
    'last_name' => 'Smith',
    'email' => 'jane@example.com',
]);
```

### Using the Main API Client Class

```php
require_once __DIR__ . '/api-client.php';

// Local API
$api = new WhmcsApiClient([
    'api_url' => 'https://your-whmcs.com/includes/api.php',
    'api_key' => 'admin_username',
    'access_key' => 'admin_password',
    'api_type' => 'local',
]);

// Get clients
$clients = $api->getClients();

// Get a specific client
$client = $api->getClient(12345);

// REST API
$api = new WhmcsApiClient([
    'api_url' => 'https://your-whmcs.com/api/v1',
    'api_key' => 'your-api-key',
    'api_type' => 'rest',
]);

$products = $api->restApi('GET', '/products');
```

## Available Methods

### Client Operations

```php
$api->getClients(array $params);   // List clients with filters
$api->getClient(int $id);           // Get single client
$api->addClient(array $data);       // Create new client
$api->updateClient(int $id, array $data); // Update client
```

### Service Operations

```php
$api->getClientsProducts(int $clientId); // Get client's products
$api->getProducts(array $params);        // List products
$api->getOrders(array $params);         // List orders
```

### Invoice Operations

```php
$api->getInvoices(array $params);       // List invoices
$api->getInvoice(int $id);              // Get invoice
$api->addInvoicePayment(int $id, string $transId, float $amount); // Add payment
```

### Domain Operations

```php
$api->getDomains(array $params);       // List domains
```

### Ticket Operations

```php
$api->getTickets(array $params);       // List tickets
$api->openTicket(array $data);          // Create ticket
$api->addTicketReply(int $ticketId, string $message); // Reply to ticket
```

## Credential Management

```php
use WhmcsApi\ApiCredentials;

// Store credentials
$id = ApiCredentials::store('my_whmcs', 'https://whmcs.com', 'api_key', 'secret', 'local');

// Retrieve credentials
$creds = ApiCredentials::get('my_whmcs');

// Test connection
$connected = ApiCredentials::test('my_whmcs');

// List all credentials
$all = ApiCredentials::all();

// Delete credentials
ApiCredentials::delete('my_whmcs');
```

## Debug Mode

Enable debug logging to see all API requests and responses:

```php
$api = new WhmcsApiClient([
    'api_url' => 'https://your-whmcs.com/includes/api.php',
    'api_key' => 'admin_username',
    'access_key' => 'admin_password',
    'api_type' => 'local',
    'debug' => true,  // Enable logging
]);
```

## Error Handling

```php
try {
    $client = $api->getClient(99999);
} catch (\Exception $e) {
    echo "API Error: " . $e->getMessage();
}
```

## File Structure

```
whmcs-api-client/
├── api-client.php         # Main API client
├── lib/
│   ├── LocalApi.php       # Local API wrapper
│   ├── RestApi.php        # REST API wrapper
│   ├── ApiCredentials.php # Credential manager
│   └── ApiResponse.php    # Response handler
├── templates/
│   └── client_example.tpl # Usage templates
└── api-admin.php          # Admin interface
```

## Security Notes

- Store API credentials securely (use WHMCS encryption)
- Never expose credentials in client-side code
- Use HTTPS for all API communications
- Implement IP whitelisting when possible
- Rotate API keys regularly

## Requirements

- WHMCS 7.0+ (Local API)
- WHMCS 8.0+ (REST API)
- PHP 7.4+
- cURL extension

## Support

For issues and feature requests, please contact the developer.