# WHMCS Payment Methods

## Overview

WHMCS supports multiple payment methods including credit cards, bank transfers, e-wallets, and alternative payment options. Configuration allows for various gateway integrations and custom payment arrangements.

## Payment Method Types

### Online Gateways

| Gateway | Type | Features |
|---------|------|----------|
| Stripe | Credit Card | Subscription, ACH, Apple Pay, Google Pay |
| PayPal | Credit Card, Bank | Express Checkout, Pro, Billing |
| Authorize.net | Credit Card | CIM, eCheck |
| Braintree | Credit Card | PayPal, Venmo |
| Square | Credit Card | In-person, online |
| 2Checkout | Credit Card | Multi-currency |

### Bank Transfers

| Method | Description |
|--------|-------------|
| Direct Debit | ACH/Sepa direct debit |
| Wire Transfer | Manual bank transfer |
| Virtual Account | Auto-generate bank account |
| Crypto | Bitcoin, Ethereum, etc. |

### E-Wallets

| Wallet | Description |
|--------|-------------|
| Skrill | Online payments |
| Neteller | Gaming/prepaid |
| Paytm | India payments |
| Alipay | China payments |

## Configuring Payment Gateways

### Gateway Setup

**Configuration > Payments > Payment Gateways**

```php
// Gateway configuration
[
    'name' => 'stripe',
    'display_name' => 'Credit Card (Stripe)',
    'type' => 'card',
    'supported_currencies' => ['USD', 'EUR', 'GBP'],
    'requires_3d_secure' => true,
    'allow_recurring' => true,
    'allow_one_time' => true
]
```

### Module Activation

```php
// Activate via module hook
$params = [
    'gateway' => 'stripe',
    'api_key' => 'sk_live_xxxx',
    'publishable_key' => 'pk_live_xxxx',
    'webhook_secret' => 'whsec_xxxx'
];

$gateway = new WHMCS\\Module\\Gateway('stripe');
$gateway->activate($params);
```

## Payment Method Management

### Enable/Disable Methods

**Configuration > Payments > Payment Methods**

```php
// Available payment methods
[
    'credit_card' => [
        'enabled' => true,
        'gateways' => ['stripe', 'paypal', 'authorizenet'],
        'default' => 'stripe'
    ],
    'bank_transfer' => [
        'enabled' => true,
        'gateways' => ['wiretransfer'],
        'require_approval' => true
    ],
    'local_credit' => [
        'enabled' => true,
        'use_credit_balance' => true
    ]
]
```

### Payment Method Order

```php
// Display order in client area
[
    'order' => [
        'stripe',
        'paypal',
        'bank_transfer',
        'credit'
    ]
]
```

## Credit Card Processing

### Card Types Accepted

```php
// Card type configuration
[
    'visa' => true,
    'mastercard' => true,
    'amex' => true,
    'discover' => true,
    'jcb' => false,
    'diners' => false
]
```

### Card Security

```php
// 3D Secure / SCA
[
    'require_3ds' => true,
    'fallback_enabled' => false,
    'authentication_failed_action' => 'decline'  // or 'allow'
]

// Card verification
[
    'require_cvv' => true,
    'require_postal_code' => true,
    'require_zip_verify' => false
]
```

### Tokenization

```php
// Secure card storage
[
    'tokenize_cards' => true,
    'token_provider' => 'stripe',  // Gateway handles token
    'allow_save_cards' => true,
    'max_saved_cards' => 5
]
```

## Bank Transfer Setup

### Wire Transfer

```php
// Bank details configuration
[
    'bank_name' => 'Bank of America',
    'account_name' => 'Your Company Inc',
    'account_number' => '****1234',
    'routing_number' => '****5678',
    'swift_code' => 'BOFAUS3N',
    'iban' => 'US****',
    'instructions' => 'Include invoice number with payment'
]
```

### Virtual Account Numbers

```php
// Auto-generate virtual accounts
[
    'provider' => 'go_cardless',
    'bank' => ' deutsche_bank',
    'auto_assign' => true,
    'format' => 'GBxxYYYYYYYYYYYY'
]
```

### SEPA Direct Debit

```php
// European payments
[
    'enabled' => true,
    'countries' => ['DE', 'FR', 'ES', 'IT', 'NL'],
    'mandate_required' => true,
    'days_before_charge' => 5,
    'retry_failed' => 3
]
```

## Alternative Payment Methods

### Local Payment Methods

```php
// Region-specific
[
    'ideal' => ['Netherlands' => true],
    'bancontact' => ['Belgium' => true],
    'giropay' => ['Germany' => true],
    'sofort' => ['DE', 'AT', 'BE', 'NL' => true],
    'boleto' => ['Brazil' => true],
    'alipay' => ['China' => true]
]
```

### Crypto Currency

```php
// Cryptocurrency payments
[
    'provider' => 'bitpay',
    'coins' => ['BTC', 'ETH', 'USDT'],
    'conversion_currency' => 'USD',
    'invoice_timeout' => 60,  // minutes
    'network_confirmations' => 3
]
```

## Payment Method Routing

### Conditional Routing

```php
// Route based on conditions
[
    'rules' => [
        ['amount_min' => 1000, 'gateway' => 'bank_transfer'],
        ['amount_max' => 10, 'gateway' => 'local_credit'],
        ['country' => 'US', 'gateway' => 'stripe'],
        ['country' => 'DE', 'gateway' => 'sepa']
    ]
]
```

### Amount-Based Routing

```php
// High value routing
[
    'threshold_high' => 10000,
    'gateway_high' => 'bank_transfer',
    'require_approval' => true
]

// Low value routing
[
    'threshold_low' => 5,
    'gateway_low' => 'credit',
    'minimum_auto' => 1.00
]
```

## Recurring Payment Methods

### Subscription Setup

```php
// Recurring billing configuration
[
    'allow_subscriptions' => true,
    'default_cycle' => 'monthly',
    'auto_renew' => true,
    'retry_failed' => 3,
    'retry_days' => [1, 3, 7],
    'payment_method_update' => true
]
```

### Payment Retry

```php
// Failed payment handling
[
    'max_retries' => 3,
    'retry_schedule' => [
        'day_1' => 'first_attempt',
        'day_4' => 'second_attempt',
        'day_10' => 'final_attempt'
    ],
    'suspension_on_failure' => false,
    'notify_before_retry' => true
]
```

## Client Payment Methods

### Stored Payment Methods

**Client Area > Payment Methods**

```php
// Client's saved payment methods
[
    ['type' => 'card', 'last4' => '4242', 'brand' => 'Visa', 'default' => true],
    ['type' => 'card', 'last4' => '5555', 'brand' => 'Mastercard', 'default' => false],
    ['type' => 'bank', 'last4' => '6789', 'bank' => 'Chase', 'default' => false]
]
```

### Add Payment Method

```php
// Add new method
[
    'type' => 'card',
    'token' => 'tok_xxxx',
    'set_default' => true,
    'billing_address' => [
        'line1' => '123 Main St',
        'city' => 'New York',
        'state' => 'NY',
        'postal_code' => '10001',
        'country' => 'US'
    ]
]
```

## Payment Method Fees

### Gateway Fees

```php
// Configure per gateway
[
    'stripe' => [
        'percentage_fee' => 2.9,
        'fixed_fee' => 0.30,
        'currency_fee' => 0
    ],
    'paypal' => [
        'percentage_fee' => 3.49,
        'fixed_fee' => 0.30
    ]
]
```

### Fee Display

```php
// Show fees to customer
[
    'show_fees' => true,
    'fee_calculation' => 'real_time',
    'fee_cap' => 10.00
]
```

## API Integration

```php
// Get available payment methods
$result = localAPI('GetPaymentMethods', [
    'userid' => 123,
    'amount' => 100.00,
    'currency' => 'USD'
]);

// Charge with specific method
$result = localAPI('ChargeInvoice', [
    'invoiceid' => 5678,
    'paymentmethod' => 'stripe',
    'card_token' => 'tok_xxxx'
]);
```

## Hooks

```php
// Hook: PaymentMethodPreCharge
add_hook('PaymentMethodPreCharge', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['gateway']
    // $vars['amount']
    
    // Modify or validate before charge
});

// Hook: PaymentMethodChargeSuccess
add_hook('PaymentMethodChargeSuccess', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['transaction_id']
    // $vars['amount']
});
```

## Best Practices

1. **Multiple options**: Offer several payment methods
2. **Security first**: Enable 3D Secure and fraud protection
3. **Clear instructions**: Provide payment details clearly
4. **Mobile-friendly**: Ensure payment forms work on mobile
5. **Regular testing**: Test payment flows regularly

## Related Documentation

- [Gateway Fees](./whmcs-gateway-fees.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Payment Allocation](./whmcs-payment-allocation.md)
- [Invoice Templates](./whmcs-invoice-templates.md)