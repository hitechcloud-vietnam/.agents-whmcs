# WHMCS Client Addresses

## Overview

Client addresses in WHMCS store billing and shipping address information for clients. Multiple addresses can be stored per client with one designated as the default billing address.

## Address Management

### Accessing Addresses

**Admin: Clients > Select Client > Addresses**

### Address Types

```php
// Address types
[
    'billing' => 'Primary billing address',
    'shipping' => 'Shipping/delivery address',
    'registered' => 'Company registered address',
    'other' => 'Additional address'
]
```

## Address Fields

### Standard Address Data

```php
// Address fields
[
    'address_type' => 'billing',
    'address1' => '123 Main Street',
    'address2' => 'Suite 100',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US',
    'is_default' => true
]
```

## Multiple Addresses

### Add New Address

```php
// Add additional address
[
    'userid' => 123,
    'address_type' => 'shipping',
    'address_label' => 'Warehouse Address',
    'address1' => '456 Industrial Blvd',
    'city' => 'Los Angeles',
    'state' => 'CA',
    'postcode' => '90001',
    'country' => 'US',
    'is_default' => false
]
```

### Edit Address

```php
// Update address
[
    'address_id' => 789,
    'address1' => '123 Main Street',
    'address2' => 'Floor 5',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10002',
    'country' => 'US'
]
```

### Delete Address

```php
// Remove address
[
    'address_id' => 789,
    'userid' => 123,
    'reason' => 'No longer used'
]
```

## Default Address

### Set Default Address

```php
// Mark as default billing
[
    'address_id' => 789,
    'userid' => 123,
    'is_default' => true,
    'address_type' => 'billing'
]
```

### Auto-Use Default

```php
// Use default for invoices
[
    'use_default_billing' => true,
    'default_for_new_orders' => true
]
```

## Address Validation

### Country-Specific Validation

```php
// Validate based on country
[
    'country' => 'US',
    'require_state' => true,
    'state_format' => '2_letter',
    'postcode_format' => '#####'
]

[
    'country' => 'UK',
    'require_state' => false,
    'postcode_format' => 'AA# #AA'
]
```

## Address Display

### Invoice Address

```php
// Display on invoice
[
    'show_on_invoice' => true,
    'format' => 'block',
    'include_company' => true
]
```

### Order Address

```php
// Address on order form
[
    'show_address' => true,
    'allow_selection' => true,
    'allow_new_address' => true
]
```

## API Functions

```php
// Get client addresses
$result = localAPI('GetClientAddresses', [
    'clientid' => 123
]);

// Add address
$result = localAPI('AddClientAddress', [
    'clientid' => 123,
    'address1' => '123 Main St',
    'city' => 'New York',
    'country' => 'US'
]);

// Update address
$result = localAPI('UpdateClientAddress', [
    'addressid' => 789,
    'address1' => '123 Main Street'
]);
```

## Best Practices

1. **Validate addresses**: Ensure correct format per country
2. **Set defaults**: Designate default billing address
3. **Multiple addresses**: Support multiple delivery addresses
4. **Update regularly**: Keep addresses current
5. **Verify accuracy**: Validate addresses for shipping

## Related Documentation

- [Client Creation](./whmcs-client-creation.md)
- [Client Contacts](./whmcs-client-contacts.md)
- [Invoice Templates](./whmcs-invoice-templates.md)
- [Tax Rules](./whmcs-tax-rules.md)