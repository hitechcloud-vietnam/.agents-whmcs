# WHMCS Stripe Connect Workflow

## Overview
This workflow implements Stripe Connect for payment processing in WHMCS.

## Prerequisites
- WHMCS with Stripe integration
- Stripe Connect account
- Admin access

## Step-by-Step Process

### Step 1: Stripe Connect Setup
```php
<?php
// /includes/gateways/stripe_connect.php

class StripeConnect {
    private $secretKey;
    private $clientId;

    public function __construct()
    {
        $this->secretKey = getConfig('stripe_secret_key');
        $this->clientId = getConfig('stripe_client_id');
    }

    /**
     * Create Connect account
     */
    public function createConnectAccount(int $clientId): array
    {
        $client = getClientsDetails($clientId);

        $ch = curl_init('https://api.stripe.com/v1/accounts');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'type' => 'express',
                'email' => $client['email'],
                'metadata' => ['whmcs_client_id' => $clientId]
            ]),
            CURLOPT_USERPWD => $this->secretKey,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }

    /**
     * Create payment intent
     */
    public function createPaymentIntent(float $amount, string $currency, int $accountId): array
    {
        $ch = curl_init('https://api.stripe.com/v1/payment_intents');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'amount' => $amount * 100,
                'currency' => $currency,
                'transfer_data' => ['destination' => $accountId],
                'application_fee_amount' => $amount * 0.029 + 30
            ]),
            CURLOPT_USERPWD => $this->secretKey,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

## Related Workflows
- [WHMCS Payment Gateway Setup](./whmcs-payment-gateway-setup.md)