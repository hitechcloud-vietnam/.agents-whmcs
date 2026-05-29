# WHMCS Payment Gateway Integration Workflow

## Description
Comprehensive workflow for integrating payment gateways into WHMCS.

## Prerequisites
- Payment gateway account/API credentials
- SSL certificate
- WHMCS 7.0+

## Steps

### Step 1: Research Gateway Requirements
```markdown
Before starting, gather:
- API documentation
- Sandbox/test credentials
- Production credentials
- Webhook/IPN requirements
- Supported currencies
- Transaction types (one-time, recurring, refund)
```

### Step 2: Create Gateway Structure
```
modules/gateways/
└── clicodes_gateway/
    ├── clicodes_gateway.php
    ├── callback.php
    └── logo.png
```

### Step 3: Implement Gateway Functions
```php
<?php
// clicodes_gateway.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway metadata
 */
function clicodes_gateway_MetaData()
{
    return [
        'DisplayName' => 'CLICodes Payment',
        'APIVersion' => '1.0',
    ];
}

/**
 * Gateway configuration
 */
function clicodes_gateway_config()
{
    return [
        'FriendlyName' => ['value' => 'CLICodes Payment'],
        'merchantId' => ['Type' => 'text', 'Label' => 'Merchant ID'],
        'apiKey' => ['Type' => 'password', 'Label' => 'API Key'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,production'],
    ];
}

/**
 * Payment link
 */
function clicodes_gateway_link($params)
{
    // Build and return payment form
    $html = '<form action="...">...</form>';
    return $html;
}
```

### Step 4: Implement Callback Handler
```php
<?php
// callback.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('clicodes_gateway');

// Verify webhook signature
// Process payment
// Log transaction
// Add invoice payment

header('Location: ' . $systemurl . '/viewinvoice.php?id=' . $invoiceId);
```

### Step 5: Test Sandbox
```bash
# Configure gateway in test mode
# Create test invoice
# Process test payment
# Verify callback received
# Check invoice marked as paid
```

### Step 6: Production Deployment
```bash
# Switch to production credentials
# Test with small amount
# Monitor transactions
# Enable monitoring
```

## Gateway Types
| Type | Description | Example |
|------|-------------|---------|
| Hosted | Redirect to gateway | PayPal, Stripe |
| Direct | Collect on-site | Authorize.Net |
| Token | Store tokens | Braintree |
| Local | Manual processing | Bank Transfer |

## Transaction Flow
```
1. Client selects payment method
2. WHMCS creates invoice
3. Gateway link displayed
4. Client redirected to gateway
5. Payment processed
6. Gateway sends webhook
7. WHMCS updates invoice
8. Service provisioned
```

## Tags
- payment
- gateway
- integration
- transaction