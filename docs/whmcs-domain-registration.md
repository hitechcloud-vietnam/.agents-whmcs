# WHMCS Domain Registration

## Overview

Domain registration in WHMCS automates the process of registering domain names through ICANN-accredited registrars. This includes TLD selection, registrant data, and DNS configuration.

## Registration Process

### Client Order Flow

**Client Area > Order Domain**

```php
// Domain registration order
[
    'domain' => 'example.com',
    'tld' => 'com',
    'registration_period' => 1,       // years
    'registrant' => [...],
    'nameservers' => ['ns1.host.com', 'ns2.host.com']
]
```

### Registration Steps

1. Check domain availability
2. Validate domain syntax
3. Collect registrant data
4. Process payment
5. Register with registrar
6. Configure DNS
7. Send confirmation

## Domain Pricing

### TLD Configuration

**Configuration > Domains > Pricing**

```php
// TLD pricing
[
    'tld' => 'com',
    'registration' => [
        '1_year' => 9.95,
        '2_years' => 19.90,
        '5_years' => 49.75
    ],
    'transfer' => 9.95,
    'renewal' => 9.95
]
```

## Registrant Data

### WHOIS Information

```php
// Registrant details
[
    'registrant_name' => 'John Doe',
    'registrant_organization' => 'Example Inc',
    'registrant_email' => 'john@example.com',
    'registrant_address' => '123 Main St',
    'registrant_city' => 'New York',
    'registrant_state' => 'NY',
    'registrant_postcode' => '10001',
    'registrant_country' => 'US',
    'registrant_phone' => '+1.5550100'
]
```

## Nameserver Configuration

### Set Nameservers

```php
// Configure DNS
[
    'domain_id' => 1,
    'nameservers' => [
        'ns1.yourcompany.com',
        'ns2.yourcompany.com',
        'ns3.yourcompany.com'     // optional
    ]
]
```

## Domain Status

### Status Types

| Status | Description |
|--------|-------------|
| Pending | Registration in progress |
| Active | Registered and active |
| Expired | Registration expired |
| Transferred Away | Transferred to another registrar |

## Auto-Registration

### Module Configuration

```php
// Registrar module settings
[
    'module' => 'enom',
    'api_key' => 'xxx',
    'auto_register' => true,
    'registration_delay' => 0
]
```

## Registration Confirmation

### Client Notification

```smarty
Subject: Domain Registered - {$domain}

Dear {$client_name},

Your domain has been registered successfully.

Domain: {$domain}
Expiration: {$expiry_date}
Registrant: {$registrant}

DNS Nameservers:
- {$nameserver_1}
- {$nameserver_2}

{$company_name}
```

## API Functions

```php
// Check availability
$result = localAPI('CheckDomain', [
    'domain' => 'example.com'
]);

// Register domain
$result = localAPI('RegisterDomain', [
    'domain' => 'example.com',
    'years' => 1
]);
```

## Hooks

```php
// Hook: DomainRegistered
add_hook('DomainRegistered', 1, function($vars) {
    // $vars['domainid']
    // $vars['domain']
    // Configure DNS, notify, etc.
});
```

## Best Practices

1. **TLD coverage**: Offer popular TLDs
2. **Competitive pricing**: Research registrar prices
3. **Auto-renew**: Enable automatic renewals
4. **Clear WHOIS**: Ensure accurate registrant data
5. **Confirmation**: Send timely notifications

## Related Documentation

- [Domain Transfer](./whmcs-domain-transfer.md)
- [Domain Renewal](./whmcs-domain-renewal.md)
- [Domain Pricing](./whmcs-domain-pricing.md)
- [Domain Sync](./whmcs-domain-sync.md)