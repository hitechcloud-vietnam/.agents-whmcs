# WHMCS Module Parameters

Complete reference for parameters passed to WHMCS module functions.

## Overview

Module functions receive standardized parameters that provide context about the service, client, and configuration.

## Common Parameters

### Service Parameters

All module functions receive service context parameters:

```php
/**
 * Standard service parameters passed to module functions
 */
$params = [
    // Service Information
    'serviceid' => 123,                    // Service ID
    'domain' => 'example.com',            // Service domain
    'username' => 'exampleuser',          // Service username
    'password' => 'encrypted_pass',       // Encrypted password
    'domainid' => 456,                    // Domain ID (if applicable)
    
    // Product/Service Details
    'packageid' => 1,                     // Product ID
    'configoption1' => 'value1',         // Configurable option 1
    'configoption2' => 'value2',         // Configurable option 2
    'configoption3' => 'value3',         // Configurable option 3
    // ... up to configoption20
    
    // Client Information
    'clientsdetails' => [
        'id' => 789,
        'email' => 'client@example.com',
        'firstname' => 'John',
        'lastname' => 'Doe',
        'companyname' => 'Acme Corp',
        'address1' => '123 Main St',
        'city' => 'Anytown',
        'state' => 'ST',
        'postcode' => '12345',
        'country' => 'US',
        'phonenumber' => '+1-555-1234',
    ],
    
    // Server Information
    'serverid' => 5,                      // Server ID
    'serverip' => '192.168.1.100',       // Server IP
    'serverhostname' => 'server1.example.com',
    'serverusername' => 'root',         // Server username
    'serverpassword' => 'encrypted',     // Server password
    'serveraccesshash' => 'access_hash',
    
    // Module Configuration
    'module' => 'module_name',
    'configoptions' => [                 // Selected config options
        1 => 'value1',
        2 => 'value2',
    ],
];
```

### Server Module Parameters

Parameters specific to server/provisioning modules:

```php
/**
 * Server module specific parameters
 */
$serverParams = [
    // Server Connection
    'serverid' => 5,
    'serverip' => '192.168.1.100',
    'serverhostname' => 'server.example.com',
    'serverusername' => 'root',
    'serverpassword' => 'encrypted_server_pass',
    'serveraccesshash' => 'access_hash',
    'serversecure' => true,              // Use SSL
    
    // Module Configuration
    'configoption1' => 'module_setting_1',
    'configoption2' => 'module_setting_2',
    // ... through configoption20
    
    // Service Details
    'serviceid' => 123,
    'domain' => 'example.com',
    'username' => 'service_username',
    'password' => 'service_password',
    'packageid' => 1,
    
    // Suspend Reason (for suspend function)
    'suspendreason' => 'Payment overdue',
    
    // Custom Fields
    'customfields' => [
        'Field Name' => 'value',
    ],
    'configoptions' => [
        // Config option values
    ],
];
```

### Gateway Module Parameters

Parameters for payment gateway modules:

```php
/**
 * Gateway module specific parameters
 */
$gatewayParams = [
    // Invoice Details
    'invoiceid' => 1001,
    'invoicenum' => 'INV-2024-0001',
    'amount' => 99.99,
    'currency' => 'USD',
    'total' => 99.99,
    'tax' => 0.00,
    
    // Client Details
    'clientdetails' => [
        'id' => 789,
        'email' => 'client@example.com',
        'firstname' => 'John',
        'lastname' => 'Doe',
        'companyname' => 'Acme Corp',
    ],
    
    // Order Details
    'orderid' => 123,
    'ordernumber' => 'ORD-2024-0001',
    
    // Configuration
    'config' => [
        'apikey' => 'your_api_key',
        'merchantid' => 'merchant_123',
    ],
    
    // System URLs
    'systemurl' => 'https://whmcs.example.com',
    'returnurl' => 'https://whmcs.example.com/viewinvoice.php?id=1001',
    
    // Transaction
    'transactionid' => 'txn_123456',      // For refund
    'refundreason' => 'Customer request', // For refund
];
```

### Registrar Module Parameters

Parameters for domain registrar modules:

```php
/**
 * Registrar module specific parameters
 */
$registrarParams = [
    // Domain Components
    'sld' => 'example',                   // Second-level domain
    'tld' => '.com',                      // Top-level domain
    'domain' => 'example.com',           // Full domain name
    
    // Registration Period
    'regperiod' => 1,                    // Years
    
    // Transfer Secret (for transfer)
    'transfersecret' => 'auth_code_123',
    
    // Registrant Information
    'firstname' => 'John',
    'lastname' => 'Doe',
    'companyname' => 'Acme Corp',
    'email' => 'client@example.com',
    'address1' => '123 Main St',
    'address2' => '',
    'city' => 'Anytown',
    'state' => 'ST',
    'postcode' => '12345',
    'country' => 'US',
    'phonenumber' => '+1.5551234567',
    
    // Nameservers
    'ns1' => 'ns1.example.com',
    'ns2' => 'ns2.example.com',
    'ns3' => 'ns3.example.com',
    'ns4' => 'ns4.example.com',
    
    // Domain ID (for existing domains)
    'domainid' => 456,
    'domainid' => 456,
    
    // DNS Records (for DNS management)
    'dnsrecords' => [
        ['name' => '@', 'type' => 'A', 'address' => '192.168.1.1'],
    ],
    
    // Contacts
    'contacts' => [
        'Registrant' => [...],
        'Admin' => [...],
        'Tech' => [...],
        'Billing' => [...],
    ],
    
    // Lock Status
    'lockenabled' => 'locked',
    
    // Module Configuration
    'configoption1' => 'setting1',
    'configoption2' => 'setting2',
];
```

## Accessing Parameters

### In Module Functions

```php
/**
 * Example: Access parameters in provisioning module
 */
function yourmodule_CreateAccount(array $params)
{
    // Access service details
    $domain = $params['domain'];
    $username = $params['username'];
    
    // Access client email
    $clientEmail = $params['clientsdetails']['email'];
    
    // Access module config
    $apiKey = $params['configoption1'];
    
    // Access server details
    $serverIp = $params['serverip'];
    
    // Access config options
    $selectedOption = $params['configoptions'][5] ?? 'default';
}
```

### Parameter Access Helpers

```php
/**
 * Get parameter with default value
 */
function getParam(array $params, string $key, $default = null)
{
    return $params[$key] ?? $default;
}

/**
 * Get nested parameter
 */
function getNestedParam(array $params, string $key1, string $key2, $default = null)
{
    return $params[$key1][$key2] ?? $default;
}
```

## Parameter Types

| Type | Description |
|------|-------------|
| serviceid | Integer service ID |
| domain | String domain name |
| username | String service username |
| password | Encrypted string password |
| clientsdetails | Array of client info |
| server* | Server connection parameters |
| configoption* | Module configuration values |
| configoptions | Selected config options array |

## Related Documentation

- [whmcs-module-provisioning-api.md](whmcs-module-provisioning-api.md)
- [whmcs-module-gateway-api.md](whmcs-module-gateway-api.md)
- [whmcs-module-registrar-api.md](whmcs-module-registrar-api.md)