# WHMCS Currency Conversion

## Overview

Currency conversion in WHMCS enables businesses to display prices and accept payments in multiple currencies. The system handles exchange rate management, conversion calculations, and proper display of pricing across currencies.

## Multi-Currency Configuration

### Enabling Multi-Currency

**Configuration > General > Currency Settings**

```php
[
    'multi_currency_enabled' => true,
    'base_currency' => 'USD',
    'allow_client_currency_selection' => true,
    'default_client_currency' => 'USD'
]
```

## Currency Setup

### Adding Currencies

**Configuration > Currencies**

```php
// Currency configuration
[
    'code' => 'EUR',
    'name' => 'Euro',
    'symbol' => '€',
    'format' => '1,234.56',
    'base' => false,
    'rate' => 1.10,        // Exchange rate to base currency
    'updated_at' => '2024-05-15'
]
```

### Currency Parameters

| Field | Description | Example |
|-------|-------------|---------|
| Code | ISO currency code | EUR |
| Name | Full name | Euro |
| Symbol | Currency symbol | € |
| Format | Display format | 1,234.56 |
| Rate | Exchange rate | 1.10 |
| Base | Primary currency | false |

## Exchange Rate Management

### Automatic Updates

**Configuration > Currencies > Update Rates**

```php
// Exchange rate provider
[
    'provider' => 'openexchangerates',  // or fixer, xe, etc.
    'api_key' => 'your_api_key',
    'update_frequency' => 'daily',      // hourly, daily, weekly
    'last_update' => '2024-05-15 00:00:00',
    'auto_update' => true
]
```

### Manual Rate Setting

```php
// Manual rate entry
[
    'code' => 'EUR',
    'rate' => 1.10,
    'updated_by' => 'admin_id',
    'updated_at' => '2024-05-15'
]
```

### Rate Rounding

```php
// Configure rounding
[
    'decimal_places' => 2,
    'rounding_mode' => 'nearest',      // nearest, up, down
    'minimum_amount' => 0.01
]
```

## Conversion Calculations

### Basic Conversion Formula

```php
function convertCurrency($amount, $fromCurrency, $toCurrency) {
    // Get rates
    $fromRate = getExchangeRate($fromCurrency);
    $toRate = getExchangeRate($toCurrency);
    
    // Convert to base, then to target
    $baseAmount = $amount / $fromRate;
    $convertedAmount = $baseAmount * $toRate;
    
    return round($convertedAmount, 2);
}
```

### Example Conversions

```php
// USD to EUR (rate: 1.10)
$amount = 100.00;
$converted = 100.00 * 1.10;  // €110.00

// EUR to USD (rate: 0.9091)
$amount = 100.00;
$converted = 100.00 / 1.10;  // $90.91

// EUR to GBP (rate: 0.85)
$amount = 100.00;
$baseUSD = 100.00 / 1.10;    // $90.91
$convertedGBP = 90.91 * 0.85; // £77.27
```

## Product Pricing by Currency

### Base Pricing

```php
// Product configured in base currency
[
    'product_id' => 1,
    'name' => 'Basic Hosting',
    'pricing' => [
        'USD' => ['monthly' => 10.00],
        'EUR' => ['monthly' => 9.00],
        'GBP' => ['monthly' => 8.00]
    ]
]
```

### Dynamic Pricing

```php
// Calculate from base currency
[
    'use_conversion' => true,
    'markup_percentage' => 0,
    'minimum_price' => 5.00
]

// EUR: 10.00 * 1.10 = 11.00
// GBP: 10.00 * 0.85 = 8.50
```

### Manual Override

```php
// Set specific prices per currency
[
    'USD' => 10.00,
    'EUR' => 11.00,      // Manually set
    'GBP' => 8.50        // Manually set
]
```

## Client Currency

### Setting Client Currency

```php
// Client currency preference
[
    'client_id' => 123,
    'currency' => 'EUR',
    'preferred_payment_currency' => 'EUR'
]
```

### Currency Change

```php
// Allow client to change currency
// Client area setting
[
    'allow_currency_change' => true,
    'allowed_currencies' => ['USD', 'EUR', 'GBP']
]
```

## Invoice Currency

### Invoice Display

```php
// Invoice in client's currency
[
    'invoice_number' => 'INV-1234',
    'currency' => 'EUR',
    'exchange_rate' => 1.10,
    'items' => [
        ['description' => 'Hosting', 'amount' => 11.00]
    ],
    'total' => 11.00,
    'total_base_currency' => 10.00
]
```

### Currency on Payment

```php
// Payment in different currency
[
    'invoice_currency' => 'EUR',
    'invoice_amount' => 110.00,
    'payment_currency' => 'USD',
    'payment_amount' => 100.00,
    'exchange_rate_used' => 1.10
]
```

## Currency Formatting

### Display Formats

```php
// Configure per currency
[
    'USD' => [
        'symbol' => '$',
        'format' => '$1,234.56',
        'decimal_places' => 2,
        'decimal_separator' => '.',
        'thousand_separator' => ','
    ],
    'EUR' => [
        'symbol' => '€',
        'format' => '€1.234,56',
        'decimal_places' => 2,
        'decimal_separator' => ',',
        'thousand_separator' => '.'
    ]
]
```

### Template Display

```smarty
// Display currency properly
{$price|currency:$currency_code}
<!-- Outputs: €10,00 -->
```

## Payment Processing

### Multi-Currency Payments

```php
// Accept payment in invoice currency
[
    'invoice_currency' => 'EUR',
    'invoice_total' => 110.00,
    'payment_method' => 'stripe',
    'payment_currency' => 'EUR',
    'auto_convert' => true
]
```

### Settlement Currency

```php
// Convert to base currency for settlement
[
    'received_currency' => 'EUR',
    'received_amount' => 110.00,
    'exchange_rate' => 1.10,
    'settlement_amount' => 100.00
]
```

## Reports and Analytics

### Multi-Currency Reports

**Reports > Billing > Currency Report**

```php
// Revenue by currency
[
    'period' => 'May 2024',
    'currencies' => [
        'USD' => [
            'revenue' => 50000.00,
            'invoices' => 450,
            'transactions' => 500
        ],
        'EUR' => [
            'revenue' => 25000.00,
            'invoices' => 200,
            'transactions' => 220
        ],
        'GBP' => [
            'revenue' => 15000.00,
            'invoices' => 150,
            'transactions' => 160
        ]
    ],
    'total_base_currency' => 90000.00
]
```

### Exchange Rate Variance

```php
// Track rate differences
[
    'invoice_date' => '2024-05-01',
    'invoice_currency' => 'EUR',
    'invoice_amount' => 110.00,
    'rate_used' => 1.10,
    'current_rate' => 1.12,
    'variance' => 0.02
]
```

## API Functions

```php
// Convert amount
$params = [
    'amount' => 100.00,
    'from' => 'USD',
    'to' => 'EUR'
];
$result = localAPI('ConvertCurrency', $params);

// Get exchange rates
$params = [
    'currencies' => ['EUR', 'GBP', 'CAD']
];
$result = localAPI('GetExchangeRates', $params);

// Update currency rates
$params = [
    'code' => 'EUR',
    'rate' => 1.10
];
$result = localAPI('UpdateCurrencyRate', $params);
```

## Hooks

```php
// Hook: CurrencyConvert
add_hook('CurrencyConvert', 1, function($vars) {
    // $vars['amount']
    // $vars['from_currency']
    // $vars['to_currency']
    // $vars['converted_amount']
    
    // Modify conversion if needed
    return ['converted_amount' => $vars['converted_amount']];
});

// Hook: ExchangeRateUpdated
add_hook('ExchangeRateUpdated', 1, function($vars) {
    // $vars['currency']
    // $vars['old_rate']
    // $vars['new_rate']
});
```

## Best Practices

1. **Stable base currency**: Use the most common currency
2. **Regular updates**: Keep exchange rates current
3. **Clear display**: Show currency with amounts
4. **Payment handling**: Ensure gateway supports currencies
5. **Reporting**: Track revenue in base currency

## Related Documentation

- [Multi-Currency](./whmcs-multi-currency.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Gateway Fees](./whmcs-gateway-fees.md)
- [Invoice Generation](./whmcs-invoice-generation.md)