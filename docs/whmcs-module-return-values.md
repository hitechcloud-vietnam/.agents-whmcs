# WHMCS Module Return Values

Complete reference for return values from WHMCS module functions.

## Overview

Module functions must return specific array structures for WHMCS to interpret results correctly.

## Success Returns

### Basic Success

```php
/**
 * Basic success return
 */
return [
    'success' => true,
];
```

### Success with Transaction ID

```php
/**
 * Success with transaction reference
 */
return [
    'success' => true,
    'transactionid' => 'txn_123456789',
];
```

### Success with Account ID

```php
/**
 * Success with account ID
 */
return [
    'success' => true,
    'accountid' => 456, // Service ID or custom reference
];
```

### Success with Raw Data

```php
/**
 * Success with additional data
 */
return [
    'success' => true,
    'rawdata' => [
        'server_response' => 'data',
        'additional_info' => 'value',
    ],
];
```

### Success with SSO URL

```php
/**
 * Success with single sign-on URL
 */
return [
    'success' => true,
    'redirecturl' => 'https://controlpanel.example.com/login?token=abc123',
];
```

## Error Returns

### Basic Error

```php
/**
 * Basic error return
 */
return [
    'error' => 'Something went wrong',
];
```

### Error with Error Code

```php
/**
 * Error with code for handling
 */
return [
    'error' => 'Invalid credentials',
    'errorCode' => 'AUTH_FAILED',
];
```

### Error with Raw Data

```php
/**
 * Error with API response data
 */
return [
    'error' => 'Payment declined',
    'rawdata' => [
        'decline_code' => 'insufficient_funds',
        'network_code' => 'R01',
    ],
];
```

## Special Returns

### Module Call Return

For logging module API calls:

```php
/**
 * Log module call for debugging
 */
logModuleCall(
    'yourmodule',
    'CreateAccount',
    $params,
    $response,
    $status,
    '',
    $startTime,
    microtime(true)
);
```

### Usage Update Return

```php
/**
 * Usage update success return
 */
return [
    'success' => true,
    'diskusage' => 5000,         // MB used
    'disklimit' => 10000,       // MB limit
    'bwusage' => 100,           // MB used
    'bwlimit' => 500,           // MB limit
];
```

### Domain Info Return

```php
/**
 * Domain information return
 */
return [
    'success' => true,
    'registrationdate' => '2024-01-01',
    'expirydate' => '2025-01-01',
    'domainstatus' => 'Active',
    'dnsmanage' => true,
    'emailforward' => true,
    'idprotect' => true,
];
```

### Nameserver Return

```php
/**
 * Nameserver retrieval return
 */
return [
    'success' => true,
    'ns1' => 'ns1.example.com',
    'ns2' => 'ns2.example.com',
    'ns3' => null,
    'ns4' => null,
];
```

### Lock Status Return

```php
/**
 * Registrar lock status return
 */
return [
    'success' => true,
    'lockstatus' => 'locked', // or 'unlocked'
];
```

### DNS Records Return

```php
/**
 * DNS records return
 */
return [
    'success' => true,
    'dnsrecords' => [
        [
            'name' => '@',
            'type' => 'A',
            'priority' => 0,
            'address' => '192.168.1.1',
            'ttl' => 3600,
        ],
        [
            'name' => 'www',
            'type' => 'CNAME',
            'priority' => 0,
            'address' => '@',
            'ttl' => 3600,
        ],
    ],
];
```

### Contact Details Return

```php
/**
 * Contact details return
 */
return [
    'success' => true,
    'Registrant' => [
        'First Name' => 'John',
        'Last Name' => 'Doe',
        'Company Name' => 'Acme Corp',
        'Email Address' => 'john@example.com',
        'Address 1' => '123 Main St',
        'City' => 'Anytown',
        'State' => 'ST',
        'Postcode' => '12345',
        'Country' => 'US',
        'Phone' => '+1.5551234567',
    ],
    'Admin' => [...],
    'Tech' => [...],
    'Billing' => [...],
];
```

## Gateway Specific Returns

### Capture Return

```php
/**
 * Payment capture return
 */
return [
    'success' => true,
    'transactionid' => 'ch_1234567890',
    'rawdata' => [
        'id' => 'ch_1234567890',
        'amount' => 9999,
        'currency' => 'usd',
        'status' => 'succeeded',
    ],
];
```

### Refund Return

```php
/**
 * Refund return
 */
return [
    'success' => true,
    'refundid' => 're_1234567890',
    'rawdata' => [...],
];
```

### Subscription Return

```php
/**
 * Subscription creation return
 */
return [
    'success' => true,
    'subscriptionid' => 'sub_1234567890',
    'rawdata' => [
        'id' => 'sub_1234567890',
        'status' => 'active',
        'current_period_end' => 1704067200,
    ],
];
```

## Lifecycle Returns

### Activation Return

```php
/**
 * Module activation return
 */
return [
    'status' => 'success',
    'description' => 'Module activated successfully',
    'additionaldata' => [
        'version' => '1.0',
    ],
];
```

### Deactivation Return

```php
/**
 * Module deactivation return
 */
return [
    'status' => 'success',
    'description' => 'Module deactivated',
];
```

### Upgrade Return

```php
/**
 * Module upgrade return
 */
return [
    'status' => 'success',
    'description' => 'Upgraded to version 1.2',
];
```

## Best Practices

1. **Always return arrays** - Never return strings or throw exceptions
2. **Include raw data** - Always include response data for debugging
3. **Use consistent keys** - Use 'success'/'error' not variations
4. **Log module calls** - Use logModuleCall for API interactions
5. **Include transaction IDs** - Always return reference IDs
6. **Handle all cases** - Return proper values for success and failure

## Related Documentation

- [whmcs-module-error-handling.md](whmcs-module-error-handling.md)
- [whmcs-module-provisioning-api.md](whmcs-module-provisioning-api.md)