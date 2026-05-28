# WHMCS Tokenization Gateway Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building tokenization payment gateways for WHMCS (subscriptions, stored cards).

## When to Use

- Creating subscription billing systems
- Building merchant gateways with card storage
- Implementing recurring payments

## Tokenization Gateway Pattern

```php
<?php
// modules/gateways/{module}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_config(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Tokenization Gateway'],
        'apiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'merchantId' => ['FriendlyName' => 'Merchant ID', 'Type' => 'text'],
    ];
}

function {module}_capture_form(array $params): array {
    // Return card capture form
    return [
        'type' => 'form',
        'fields' => [
            ['name' => 'card_number', 'type' => 'text', 'label' => 'Card Number'],
            ['name' => 'card_cvv', 'type' => 'text', 'label' => 'CVV'],
            ['name' => 'card_expiry', 'type' => 'text', 'label' => 'Expiry (MM/YY)'],
        ],
    ];
}

function {module}_capture_token(array $params): array {
    // Tokenize card with provider
    try {
        $token = tokenizeCard($params['post']);

        // Save token for future use
        saveCardToken($params['client_id'], $token, $params['post']);

        return [
            'token' => $token,
            'card_type' => detectCardType($params['post']['card_number']),
            'card_last' => substr($params['post']['card_number'], -4),
            'exp_date' => $params['post']['card_expiry'],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_capture(array $params): array {
    // Charge using stored token
    try {
        $result = chargeToken($params['token'], $params['amount'] * 100);

        return [
            'status' => 'success',
            'transid' => $result['transaction_id'],
        ];
    } catch (\Exception $e) {
        return ['status' => 'failed', 'error' => $e->getMessage()];
    }
}
```

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-subscription-billing
- whmcs-security-hardening