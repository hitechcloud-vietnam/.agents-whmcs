# WHMCS Premium Domains

## Overview

Premium domains in WHMCS are high-value domain names with higher registration and renewal costs. These domains often have brand value or short names.

## Premium Domain Configuration

### Mark as Premium

**Configuration > Domains > Premium Domains**

```php
// Premium domain settings
[
    'enable_premium' => true,
    'min_premium_price' => 50.00,
    'auto_detect' => false
]
```

## Premium Pricing

### Higher Costs

```php
// Premium pricing
[
    'tld' => 'com',
    'premium_registration' => 50.00,
    'premium_transfer' => 50.00,
    'premium_renewal' => 50.00
]
```

## Premium TLDs

### Common Premium TLDs

| TLD | Premium Reason |
|-----|----------------|
| .io | Tech/business |
| .co | Short, global |
| .ai | Tech/AI |
| .app | Tech |
| .dev | Tech |

## Premium Detection

### Identify Premium

```php
// Check if premium
[
    'domain' => 'premium.io',
    'is_premium' => true,
    'premium_price' => 50.00,
    'registry_premium' => true
]
```

## Ordering Premium

### Purchase Premium

```php
// Order premium domain
[
    'domain' => 'premium.io',
    'is_premium' => true,
    'registration_price' => 50.00,
    'confirm_premium' => true
]
```

## API Functions

```php
// Check premium status
$result = localAPI('CheckPremiumDomain', [
    'domain' => 'premium.io'
]);

// Get premium pricing
$result = localAPI('GetPremiumDomainPricing', [
    'tld' => 'io'
]);
```

## Best Practices

1. **Clear pricing**: Show premium costs clearly
2. **Educate customers**: Explain premium value
3. **Set minimums**: Define premium thresholds
4. **Monitor trends**: Track premium domain demand

## Related Documentation

- [Domain Pricing](./whmcs-domain-pricing.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Auction](./whmcs-domain-auction.md)