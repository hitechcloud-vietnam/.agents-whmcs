# WHMCS Multi-Currency Setup

## Overview

WHMCS multi-currency functionality enables businesses to display pricing, invoice customers, and accept payments in multiple currencies. This is essential for serving international clients and operating globally.

## Enabling Multi-Currency

### System Configuration

**Configuration > General > Currency**

```php
// Enable multi-currency
[
    'multi_currency' => [
        'enabled' => true,
        'base_currency' => 'USD',
        'auto_convert' => true,
        'allow_client_currency' => true
    ]
]
```

## Currency Management

### Adding New Currency

**Configuration > Currencies > Add Currency**

```php
// Currency fields
[
    'code' => 'EUR',           // ISO 4217 code
    'name' => 'Euro',          // Display name
    'symbol' => '€',           // Currency symbol
    'prefix' => '',            // Symbol prefix
    'suffix' => ' EUR',        // Symbol suffix
    'format' => 1,             // Format type (1=1,234.56, 2=1.234,56)
    'rate' => 1.10,            // Exchange rate to base
    'default' => false         // Default for new clients
]
```

### Currency List

| Code | Name | Symbol | Rate | Active |
|------|------|--------|------|--------|
| USD | US Dollar | $ | 1.00 | Yes |
| EUR | Euro | € | 1.10 | Yes |
| GBP | British Pound | £ | 0.85 | Yes |
| CAD | Canadian Dollar | C$ | 1.25 | Yes |
| AUD | Australian Dollar | A$ | 1.35 | Yes |

## Exchange Rate Management

### Automatic Updates

```php
// Configure rate provider
[
    'provider' => 'openexchangerates',
    'api_key' => 'your_api_key_here',
    'update_schedule' => 'daily',  // hourly, daily, weekly
    'last_update' => '2024-05-15',
    'auto_update_enabled' => true
]
```

### Provider Options

| Provider | API Type | Update Frequency |
|----------|----------|------------------|
| Open Exchange Rates | API key | Varies |
| Fixer.io | API key | Daily free |
| ExchangeRate-API | API key | Daily free |
| xe.com | Manual | Manual |
| Manual | None | Manual only |

### Manual Rate Entry

```php
// Set rates manually
[
    'currency' => 'EUR',
    'rate' => 1.10,
    'updated_by' => 'admin_id',
    'updated_at' => '2024-05-15 10:00:00'
]
```

## Client Currency Assignment

### Default Currency

```php
// System default
[
    'default_currency' => 'USD',
    'apply_to' => 'new_clients'
]
```

### Client Currency Setting

```php
// Per-client currency
[
    'client_id' => 123,
    'currency' => 'EUR',
    'payment_currency' => 'EUR',
    'created_at' => '2024-01-15'
]
```

### Change Client Currency

**Admin: Clients > Select Client > Edit > Currency**

```php
// Currency change
[
    'userid' => 123,
    'currency' => 'GBP',
    'keep_existing_invoices' => true,
    'convert_pending_orders' => false
]
```

## Product Pricing

### Pricing by Currency

**Products > Edit Product > Pricing**

```php
// Set prices per currency
[
    'monthly' => [
        'USD' => 10.00,
        'EUR' => 9.00,
        'GBP' => 8.00,
        'CAD' => 13.00,
        'AUD' => 15.00
    ],
    'annually' => [
        'USD' => 100.00,
        'EUR' => 90.00,
        'GBP' => 80.00,
        'CAD' => 130.00,
        'AUD' => 150.00
    ]
]
```

### Dynamic Pricing

```php
// Calculate from base currency
[
    'pricing_method' => 'calculated',
    'base_currency' => 'USD',
    'markup' => 0,           // percentage markup
    'minimum_price' => [
        'EUR' => 8.00,
        'GBP' => 7.00
    ],
    'maximum_price' => [
        'EUR' => 15.00
    ]
]
```

## Invoice Currency

### Invoice Generation

```php
// Invoice in client's currency
[
    'invoice_number' => 'INV-2024-0123',
    'currency' => 'EUR',
    'client_id' => 123,
    'items' => [...],
    'total' => 110.00,
    'exchange_rate' => 1.10,
    'base_currency_total' => 100.00
]
```

### Currency Display

```smarty
<!-- Invoice template display -->
<div class="invoice-currency">
    Currency: {$invoice.currency_code}
</div>

<!-- Price formatting -->
{$lineitem.amount|currency:$invoice.currency_code}
<!-- Outputs: €110,00 for EUR -->
```

## Payment Processing

### Multi-Currency Payments

```php
// Process payment in invoice currency
[
    'invoice_id' => 5678,
    'invoice_currency' => 'EUR',
    'amount' => 110.00,
    'payment_method' => 'stripe',
    'auto_capture' => true
]
```

### Gateway Currency Support

```php
// Check gateway capabilities
[
    'stripe' => ['currencies' => ['USD', 'EUR', 'GBP', 'CAD', 'AUD']],
    'paypal' => ['currencies' => ['USD', 'EUR', 'GBP', 'CAD', 'AUD']],
    'custom_gateway' => ['currencies' => ['USD', 'EUR']]
]
```

### Settlement Handling

```php
// Gateway settlement in different currency
[
    'invoice_currency' => 'EUR',
    'settlement_currency' => 'USD',
    'settlement_amount' => 100.00,
    'exchange_rate_used' => 1.10,
    'settlement_date' => '2024-05-15'
]
```

## Domain Pricing by Currency

### TLD Pricing

**Configuration > Domains > Pricing**

```php
// Domain prices per currency
[
    'com' => [
        'USD' => ['registration' => 9.95, 'transfer' => 9.95, 'renewal' => 9.95],
        'EUR' => ['registration' => 8.95, 'transfer' => 8.95, 'renewal' => 8.95],
        'GBP' => ['registration' => 7.95, 'transfer' => 7.95, 'renewal' => 7.95]
    ]
]
```

## Currency Formatting

### Display Formats

```php
// Per-currency formatting
[
    'USD' => [
        'symbol' => '$',
        'position' => 'prefix',
        'decimal_places' => 2,
        'decimal_separator' => '.',
        'thousand_separator' => ','
    ],
    'EUR' => [
        'symbol' => '€',
        'position' => 'suffix',
        'decimal_places' => 2,
        'decimal_separator' => ',',
        'thousand_separator' => '.'
    ],
    'JPY' => [
        'symbol' => '¥',
        'position' => 'prefix',
        'decimal_places' => 0,
        'decimal_separator' => '.',
        'thousand_separator' => ','
    ]
]
```

### Client Area Display

```smarty
<!-- Display formatted price -->
{$product.pricing.monthly|currency:"EUR"}
<!-- Outputs: €9,00 -->

<!-- Format helper function -->
{currency_format amount=$amount currency=$currency}
```

## Reports and Analytics

### Revenue Reports

**Reports > Billing > Revenue by Currency**

```php
// Report structure
[
    'period' => 'May 2024',
    'base_currency' => 'USD',
    'revenue' => [
        'USD' => ['amount' => 50000.00, 'count' => 450],
        'EUR' => ['amount' => 25000.00, 'count' => 250],
        'GBP' => ['amount' => 15000.00, 'count' => 150]
    ],
    'total_base_equivalent' => 90000.00
]
```

### Exchange Rate Tracking

```php
// Rate history
[
    'date' => '2024-05-15',
    'currency' => 'EUR',
    'rate' => 1.10,
    'provider' => 'openexchangerates',
    'source_rate' => 1.10
]
```

## API Functions

```php
// Get available currencies
$result = localAPI('GetCurrencies');

// Get exchange rate
$params = [
    'currency' => 'EUR'
];
$result = localAPI('GetExchangeRate', $params);

// Convert amount
$params = [
    'amount' => 100.00,
    'from' => 'USD',
    'to' => 'EUR'
];
$result = localAPI('ConvertCurrency', $params);

// Update exchange rates
$result = localAPI('UpdateExchangeRates');

// Set client currency
$params = [
    'clientid' => 123,
    'currency' => 'GBP'
];
$result = localAPI('UpdateClient', $params);
```

## Hooks

```php
// Hook: CurrencyPreConvert
add_hook('CurrencyPreConvert', 1, function($vars) {
    // $vars['amount']
    // $vars['from_currency']
    // $vars['to_currency']
    
    // Modify before conversion
    return ['amount' => $vars['amount'], 'rate_override' => null];
});

// Hook: CurrencyPostConvert
add_hook('CurrencyPostConvert', 1, function($vars) {
    // $vars['original_amount']
    // $vars['converted_amount']
    // $vars['from_currency']
    // $vars['to_currency']
});

// Hook: ExchangeRateUpdate
add_hook('ExchangeRateUpdate', 1, function($vars) {
    // $vars['currency']
    // $vars['old_rate']
    // $vars['new_rate']
    // $vars['provider']
});
```

## Best Practices

1. **Limit currencies**: Only enable currencies you need
2. **Keep rates updated**: Use automatic updates
3. **Set explicit pricing**: Don't rely solely on conversion
4. **Consider settlement**: Choose currencies gateway supports
5. **Regular review**: Monitor currency usage and trends

## Configuration Checklist

- [ ] Enable multi-currency
- [ ] Set base currency
- [ ] Add required currencies
- [ ] Configure exchange rate provider
- [ ] Set up automatic updates
- [ ] Configure per-currency formatting
- [ ] Update product pricing
- [ ] Verify gateway support

## Related Documentation

- [Currency Conversion](./whmcs-currency-conversion.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Gateway Fees](./whmcs-gateway-fees.md)
- [Invoice Templates](./whmcs-invoice-templates.md)