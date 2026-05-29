# WHMCS Reseller Documentation

## Overview

Reseller documentation in WHMCS covers the setup and management of domain and hosting reseller accounts, allowing you to resell domain registrations and hosting services.

## Reseller Account Setup

### Registrar Reseller

```php
// Registrar reseller configuration
[
    'module' => 'enom',
    'reseller_id' => 'reseller123',
    'api_key' => 'xxx',
    'username' => 'reseller_user',
    'password' => 'xxx',
    'test_mode' => false
]
```

## Reseller Pricing

### Markup Configuration

```php
// Reseller margins
[
    'domain_markup' => 2.00,           // $2 above wholesale
    'renewal_markup' => 1.00,
    'transfer_markup' => 2.00,
    'id_protection_markup' => 1.00
]
```

## Domain Reselling

### Register for Customers

```php
// Resell domain registration
[
    'domain' => 'customer.com',
    'registration_price' => 11.95,      // Your price to customer
    'wholesale_cost' => 9.95,            // What you pay
    'profit' => 2.00
]
```

## Hosting Reselling

### WHM Integration

```php
// Resell WHM/cPanel
[
    'reseller_type' => 'whm',
    'reseller_account' => 'reseller_user',
    'nameservers' => ['ns1.reseller.com', 'ns2.reseller.com'],
    'auto_provision' => true
]
```

## Client Management

### Manage Resold Services

```php
// Resold domain management
[
    'domain_id' => 1,
    'registrar' => 'enom',
    'reseller_account' => 'reseller123',
    'customer_id' => 456
]
```

## Billing for Resellers

### Invoice Markup

```php
// Charge customers
[
    'domain' => 'customer.com',
    'customer_price' => 11.95,
    'wholesale_cost' => 9.95,
    'your_profit' => 2.00
]
```

## Sub-Account Management

### Create Sub-Accounts

```php
// Reseller sub-accounts
[
    'reseller_account' => 'reseller123',
    'sub_accounts' => [
        ['username' => 'sub1', 'can_register' => true],
        ['username' => 'sub2', 'can_register' => false]
    ]
]
```

## API Functions

```php
// Get reseller pricing
$result = localAPI('GetResellerPricing');

// Get reseller balance
$result = localAPI('GetResellerBalance');

// Update nameservers
$result = localAPI('UpdateResellerNameservers', [
    'ns1' => 'ns1.reseller.com',
    'ns2' => 'ns2.reseller.com'
]);
```

## Best Practices

1. **Set margins**: Define profitable pricing
2. **Monitor costs**: Track wholesale prices
3. **Stock domains**: Pre-register valuable domains
4. **Support customers**: Provide good support
5. **Sync regularly**: Keep domain data current

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Pricing](./whmcs-domain-pricing.md)
- [Domain Transfer](./whmcs-domain-transfer.md)
- [Domain Sync](./whmcs-domain-sync.md)
- [Service Creation](./whmcs-service-creation.md)