# WHMCS PayPal Integration Workflow

## Overview
This workflow implements PayPal integration for WHMCS.

## Prerequisites
- WHMCS with PayPal integration
- PayPal API credentials
- Admin access

## Step-by-Step Process

### Step 1: PayPal Integration
```php
<?php
// /includes/gateways/paypal_integration.php

class PayPalIntegration {
    private $clientId;
    private $clientSecret;
    private $mode;

    public function __construct()
    {
        $this->clientId = getConfig('paypal_client_id');
        $this->clientSecret = getConfig('paypal_client_secret');
        $this->mode = getConfig('paypal_mode') ?? 'sandbox';
    }

    /**
     * Get access token
     */
    private function getAccessToken(): string
    {
        $ch = curl_init($this->getBaseUrl() . '/oauth2/token');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => 'grant_type=client_credentials',
            CURLOPT_USERPWD => $this->clientId . ':' . $this->clientSecret,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true)['access_token'];
    }

    /**
     * Create order
     */
    public function createOrder(float $amount, string $currency): array
    {
        $accessToken = $this->getAccessToken();

        $ch = curl_init($this->getBaseUrl() . '/v2/checkout/orders');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'intent' => 'CAPTURE',
                'purchase_units' => [[
                    'amount' => [
                        'currency_code' => $currency,
                        'value' => number_format($amount, 2, '.', '')
                    ]
                ]]
            ]),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $accessToken,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }

    private function getBaseUrl(): string
    {
        return $this->mode === 'live'
            ? 'https://api-m.paypal.com'
            : 'https://api-m.sandbox.paypal.com';
    }
}
```

## Related Workflows
- [WHMCS Stripe Connect](./whmcs-stripe-connect.md)
- [WHMCS Payment Processing](./whmcs-payment-processing-workflow.md)