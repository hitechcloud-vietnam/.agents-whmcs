# WHMCS Domain ID Protection

## Overview

Domain ID protection (WHOIS privacy) in WHMCS hides the registrant's personal information from public WHOIS queries, maintaining privacy and reducing spam.

## ID Protection Configuration

### Enable Protection

**Configuration > Domains > ID Protection**

```php
// ID protection settings
[
    'enable_id_protection' => true,
    'auto_enable_new_domains' => false,
    'price' => 8.95,
    'renewal_price' => 8.95
]
```

## Protection Options

### Per-Domain Protection

```php
// Enable on domain
[
    'domain_id' => 1,
    'id_protection' => true,
    'enabled_at' => '2024-05-15',
    'proxy_registrant' => [
        'name' => 'Privacy Service',
        'organization' => 'Privacy Protected',
        'email' => 'privacy@example.com',
        'phone' => '+1.5550000'
    ]
]
```

## Privacy Data

### Proxy Information

```php
// WHOIS privacy contact
[
    'registrant_name' => 'REDACTED FOR PRIVACY',
    'registrant_organization' => 'REDACTED FOR PRIVACY',
    'registrant_email' => 'holder@proxy.com',
    'registrant_address' => '123 Privacy St',
    'registrant_city' => 'City',
    'registrant_state' => 'ST',
    'registrant_country' => 'US'
]
```

## Protection Pricing

### Privacy Costs

```php
// ID protection pricing
[
    'registration_price' => 8.95,
    'transfer_price' => 8.95,
    'renewal_price' => 8.95,
    'annual' => true                   // Annual billing
]
```

## Toggle Protection

### Enable Protection

```php
// Add privacy to domain
[
    'domain_id' => 1,
    'add_protection' => true,
    'charge_immediately' => true,
    'send_confirmation' => true
]
```

### Disable Protection

```php
// Remove privacy
[
    'domain_id' => 1,
    'remove_protection' => true,
    'reason' => 'Customer request',
    'restore_who_is' => true
]
```

## TLD Compatibility

### Supported TLDs

```php
// Check TLD support
[
    'tld' => 'com',
    'id_protection_supported' => true,
    'price' => 8.95
]

[
    'tld' => 'uk',
    'id_protection_supported' => false,
    'reason' => 'Nominet does not support'
]
```

## API Functions

```php
// Enable ID protection
$result = localAPI('EnableDomainIDProtection', [
    'domainid' => 1
]);

// Disable ID protection
$result = localAPI('DisableDomainIDProtection', [
    'domainid' => 1
]);
```

## Hooks

```php
// Hook: DomainIDProtectionChanged
add_hook('DomainIDProtectionChanged', 1, function($vars) {
    // $vars['domainid']
    // $vars['enabled']
});
```

## Best Practices

1. **Offer protection**: Make it easily available
2. **Competitive pricing**: Price reasonably
3. **Auto-enable option**: Consider auto-protecting
4. **TLD compatibility**: Check registry support
5. **Clear benefits**: Educate customers

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [WHOIS Contacts](./whmcs-domain-contacts.md)
- [Domain Privacy Policy](./whmcs-domain-privacy-police.md)
- [Domain Pricing](./whmcs-domain-pricing.md)