# WHMCS Domain Registrant

## Overview

Domain registrant management in WHMCS handles the domain owner information, including verification and updates per ICANN regulations.

## Registrant Data

### Owner Information

```php
// Registrant details
[
    'registrant_name' => 'John Doe',
    'registrant_organization' => 'Example Inc',
    'registrant_email' => 'john@example.com',
    'registrant_phone' => '+1.5550100',
    'registrant_address' => '123 Main St',
    'registrant_city' => 'New York',
    'registrant_state' => 'NY',
    'registrant_postcode' => '10001',
    'registrant_country' => 'US'
]
```

## Verification

### ICANN Verification

```php
// Registrant verification
[
    'verify_email' => true,
    'verification_required' => true,
    'verification_period_days' => 15,
    'auto_verify' => false
]
```

### Verification Process

```php
// Verify registrant
[
    'domain_id' => 1,
    'verification_sent' => '2024-05-01',
    'verification_expires' => '2024-05-16',
    'verified' => false
]
```

## Update Registrant

### Change Owner

```php
// Update registrant
[
    'domain_id' => 1,
    'new_registrant' => [...],
    'change_reason' => 'Ownership transfer',
    'approve_agreement' => true
]
```

## Registrant Verification Email

### Verification Template

```smarty
Subject: Verify Domain Registration - {$domain}

Dear {$registrant_name},

Please verify your domain registration:

Domain: {$domain}
Registrant: {$registrant_name}

Verify: {$verification_link}

{$company_name}
```

## API Functions

```php
// Get registrant info
$result = localAPI('GetDomainRegistrant', [
    'domainid' => 1
]);

// Update registrant
$result = localAPI('UpdateDomainRegistrant', [
    'domainid' => 1,
    'Registrant' => [...]
]);
```

## Best Practices

1. **Accurate data**: Keep registrant info current
2. **Verify promptly**: Complete verification quickly
3. **Monitor expiry**: Track verification deadlines
4. **Follow ICANN**: Adhere to all regulations

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Contacts](./whmcs-domain-contacts.md)
- [ID Protection](./whmcs-domain-id-protection.md)