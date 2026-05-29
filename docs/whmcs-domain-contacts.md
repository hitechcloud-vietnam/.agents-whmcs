# WHMCS Domain Contacts

## Overview

Domain contacts in WHMCS manage the WHOIS contact information for domains, including registrant, admin, tech, and billing contacts.

## Contact Types

### WHOIS Contacts

| Type | Purpose |
|------|---------|
| Registrant | Domain owner |
| Admin | Administrative contact |
| Tech | Technical contact |
| Billing | Billing contact |

## Contact Data

### Contact Information

```php
// WHOIS contact details
[
    'type' => 'registrant',
    'name' => 'John Doe',
    'organization' => 'Example Inc',
    'email' => 'john@example.com',
    'address1' => '123 Main St',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US',
    'phone' => '+1.5550100'
]
```

## Contact Management

### Update Contacts

```php
// Update WHOIS contacts
[
    'domain_id' => 1,
    'registrant' => [...],
    'admin' => [...],
    'tech' => [...],
    'billing' => [...]
]
```

### Same Contact for All

```php
// Use same for all
[
    'domain_id' => 1,
    'use_same_contact' => true,
    'contact' => [...]
]
```

## Transfer Contacts

### Update for Transfer

```php
// Contact for transfer
[
    'domain_id' => 1,
    'update_registrant' => true,
    'registrant_data' => [...],
    'auth_code' => 'ABC123'
]
```

## Contact Validation

### Required Fields

```php
// Validate contact
[
    'email' => 'valid_email',
    'phone' => 'valid_format',
    'country' => 'required',
    'address' => 'required'
]
```

## API Functions

```php
// Update domain contacts
$result = localAPI('UpdateDomainContacts', [
    'domainid' => 1,
    'Registrant' => [...],
    'Admin' => [...]
]);
```

## Best Practices

1. **Accurate data**: Keep WHOIS contacts current
2. **Valid email**: Ensure reachable email
3. **Use privacy**: Consider ID protection
4. **Follow rules**: Adhere to ICANN requirements

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain ID Protection](./whmcs-domain-id-protection.md)
- [Registrant Verification](./whmcs-domain-registrant.md)