# WHMCS Service Creation

## Overview

Service creation in WHMCS involves provisioning new hosting accounts, software licenses, and other products for clients. This process can be automated upon payment or manually initiated by admins.

## Service Creation Methods

### Automatic on Payment

```php
// Auto-provision when invoice paid
[
    'auto_setup' => true,
    'trigger' => 'invoice_paid',
    'create_service' => true,
    'send_credentials' => true
]
```

### Manual Creation

**Admin: Clients > Select Client > Add Order/Service**

```php
// Manual service creation
[
    'userid' => 123,
    'product_id' => 1,
    'billing_cycle' => 'monthly',
    'domain' => 'example.com',
    'custom_fields' => [...],
    'send_email' => true
]
```

## Product Configuration

### Product Setup Requirements

```php
// Required for provisioning
[
    'product_id' => 1,
    'product_name' => 'Web Hosting',
    'module' => 'cpanel',
    'server_group' => 1,
    'auto_setup' => true
]
```

### Configurable Options

```php
// Options at order time
[
    'config_options' => [
        ['id' => 1, 'option' => '10GB Storage', 'qty' => 1],
        ['id' => 2, 'option' => 'Daily Backups', 'selected' => true]
    ]
]
```

## Order Process

### Create Order

```php
// New order creation
[
    'client_id' => 123,
    'products' => [
        ['product_id' => 1, 'domain' => 'example.com', 'billing_cycle' => 'monthly']
    ],
    'payment_method' => 'credit_card',
    'auto_setup' => true,
    'send_email' => true
]
```

### Order Status

| Status | Description |
|--------|-------------|
| Pending | Awaiting payment |
| Active | Provisioned and running |
| Cancelled | Order cancelled |
| Fraud | Flagged for fraud check |

## Provisioning

### Module-Based Provisioning

```php
// Provision via module
[
    'module' => 'cpanel',
    'action' => 'create',
    'params' => [
        'username' => 'example',
        'domain' => 'example.com',
        'plan' => 'starter',
        'password' => 'SecurePass123!'
    ]
]
```

### Provisioning Steps

1. Validate order and payment
2. Generate username/password
3. Create on server via module
4. Configure additional options
5. Send welcome email

## Service Data

### Service Record

```php
// Service in database
[
    'id' => 1,
    'userid' => 123,
    'packageid' => 1,
    'server_id' => 1,
    'domain' => 'example.com',
    'username' => 'example',
    'password' => 'encrypted',
    'status' => 'Active',
    'created_at' => '2024-05-15',
    'next_due_date' => '2024-06-15',
    'terminate_at' => null
]
```

## Custom Fields

### Product Custom Fields

```php
// Custom field values at creation
[
    'service_id' => 1,
    'fields' => [
        ['id' => 1, 'value' => 'custom_value']
    ]
]
```

## First Invoice

### Generate First Invoice

```php
// Create first billing invoice
[
    'service_id' => 1,
    'setup_fee' => 0.00,
    'first_cycle' => 10.00,
    'prorate' => null,
    'total' => 10.00
]
```

## Welcome Email

### Send Credentials

```smarty
Subject: Your hosting account is ready

Dear {$client_name},

Your hosting account has been created!

Domain: {$service_domain}
Username: {$service_username}
Password: {$service_password}

Login: {$service_url}

{$company_name}
```

## API Functions

```php
// Create service
$result = localAPI('CreateService', [
    'clientid' => 123,
    'productid' => 1,
    'domain' => 'example.com'
]);

// Provision service
$result = localAPI('ProvisionService', [
    'serviceid' => 1
]);
```

## Best Practices

1. **Test provisioning**: Verify module works correctly
2. **Secure passwords**: Use strong generated passwords
3. **Clear communication**: Send detailed welcome emails
4. **Monitor automation**: Check auto-provision cron
5. **Backup configuration**: Save module settings

## Related Documentation

- [Service Modification](./whmcs-service-modification.md)
- [Service Cancellation](./whmcs-service-cancellation.md)
- [Product Configuration](./whmcs-product-configuration.md)
- [Module Configuration](./whmcs-module-configuration.md)