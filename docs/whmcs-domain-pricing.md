# WHMCS Domain Pricing

## Overview

Domain pricing in WHMCS configures registration, transfer, and renewal costs for TLDs (Top-Level Domains).

## TLD Pricing Configuration

### Basic Pricing

**Configuration > Domains > Pricing**

```php
// TLD pricing
[
    'tld' => 'com',
    'registration' => [
        '1_year' => 9.95,
        '2_years' => 19.90,
        '3_years' => 29.85,
        '5_years' => 49.75,
        '10_years' => 99.50
    ],
    'transfer' => 9.95,
    'renewal' => 9.95
]
```

## Pricing by Period

### Registration Periods

| Years | Price | Discount |
|-------|-------|----------|
| 1 | $9.95 | - |
| 2 | $19.90 | 0% |
| 3 | $27.00 | 10% |
| 5 | $45.00 | 10% |
| 10 | $85.00 | 15% |

## Pricing Types

### Standard Pricing

```php
// Standard registration
[
    'tld' => 'com',
    'type' => 'registration',
    'pricing' => [
        '1' => 9.95,
        '2' => 19.90
    ]
]
```

### Premium Domains

```php
// Premium TLD pricing
[
    'tld' => 'io',
    'type' => 'premium',
    'registration' => 35.00,
    'transfer' => 35.00,
    'renewal' => 35.00
]
```

## Group Pricing

### Client Group Discount

```php
// Group-specific pricing
[
    'tld' => 'com',
    'group_pricing' => [
        'default' => 9.95,
        'premium' => 8.95,
        'vip' => 7.95
    ]
]
```

## Currency Pricing

### Multi-Currency TLD Pricing

```php
// Pricing per currency
[
    'tld' => 'com',
    'currencies' => [
        'USD' => 9.95,
        'EUR' => 8.95,
        'GBP' => 7.95
    ]
]
```

## Transfer Pricing

### Transfer Costs

```php
// Transfer pricing
[
    'tld' => 'com',
    'transfer_price' => 9.95,
    'transfer_includes_renewal' => true,    // 1 year renewal included
    'registry_transfer_fee' => 0
]
```

## Renewal Pricing

### Renewal Costs

```php
// Renewal pricing
[
    'tld' => 'com',
    'renewal_price' => 9.95,
    'early_renewal_discount' => 0,
    'grace_period_renewal' => 9.95,
    'redemption_renewal' => 80.00          // During redemption period
]
```

## Bulk Pricing

### Bulk Discount

```php
// Bulk registration discount
[
    'bulk_threshold' => 5,
    'bulk_discount' => 5,                   // 5% off
    'apply_to' => ['registration', 'transfer']
]
```

## Domain Addons Pricing

### Privacy Pricing

```php
// WHOIS privacy pricing
[
    'tld' => 'com',
    'id_protection_price' => 8.95
]
```

## API Functions

```php
// Get TLD pricing
$result = localAPI('GetTLDPricing');

// Update TLD pricing
$result = localAPI('UpdateTLDPricing', [
    'tld' => 'com',
    'registration_price' => 10.95
]);
```

## Best Practices

1. **Competitive pricing**: Research market rates
2. **Bulk discounts**: Offer savings for multi-year
3. **Group pricing**: Reward loyal customers
4. **Currency support**: Price in multiple currencies
5. **Regular review**: Adjust based on costs

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Transfer](./whmcs-domain-transfer.md)
- [Domain Renewal](./whmcs-domain-renewal.md)
- [Premium Domains](./whmcs-premium-domains.md)