# WHMCS Stripe Integration Guide
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for integrating Stripe with WHMCS modules.

## When to Use

- Building Stripe-based payment gateways
- Subscription billing integration
- Stripe Connect for marketplaces

## Stripe Integration Patterns

```php
<?php
// modules/gateways/stripe/lib/StripeClient.php

namespace Stripe;

class StripeClient {
    private string $secretKey;
    private string $apiUrl = 'https://api.stripe.com/v1';

    public function __construct(string $secretKey) {
        $this->secretKey = $secretKey;
    }

    public function createCustomer(string $email, string $name): string {
        $result = $this->request('POST', '/customers', [
            'email' => $email,
            'name' => $name,
        ]);

        return $result['id'];
    }

    public function createPaymentIntent(float $amount, string $currency, string $customerId): array {
        return $this->request('POST', '/payment_intents', [
            'amount' => (int) ($amount * 100), // Stripe uses cents
            'currency' => strtolower($currency),
            'customer' => $customerId,
            'automatic_payment_methods' => ['enabled' => true],
        ]);
    }

    public function chargeCustomer(string $customerId, float $amount, string $currency): array {
        $paymentMethods = $this->request('GET', "/customers/$customerId/payment_methods");

        if (empty($paymentMethods['data'])) {
            throw new \Exception('No payment method found');
        }

        return $this->request('POST', '/payment_intents', [
            'amount' => (int) ($amount * 100),
            'currency' => strtolower($currency  ),
            'customer' => $customerId,
            'payment_method' => $paymentMethods['data'][0]['id'],
            'confirm' => true,
        ]);
    }

    public function createSubscription(string $customerId, string $priceId): array {
        return $this->request('POST', '/subscriptions', [
            'customer' => $customerId,
            'items' => [['price' => $priceId]],
        ]);
    }

    public function cancelSubscription(string $subscriptionId): array {
        return $this->request('POST', "/subscriptions/$subscriptionId", [
            'cancel_at_period_end' => true,
        ]);
    }

    private function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => $this->secretKey . ':',
            CURLOPT_TIMEOUT => 30,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($result['error']['message'] ?? 'Stripe API Error');
        }

        return $result;
    }
}
```

## Webhook Handler

```php
function stripeWebhookHandler(): void {
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_STRIPE_SIGNATURE'];

    try {
        $event = \Stripe\Webhook::constructEvent(
            $payload,
            $signature,
            $webhookSecret
        );
    } catch (\Exception $e) {
        http_response_code(400);
        exit;
    }

    switch ($event->type) {
        case 'payment_intent.succeeded':
            $paymentIntent = $event->data->object;
            // Handle successful payment
            break;

        case 'customer.subscription.created':
            // Handle new subscription
            break;

        case 'invoice.payment_failed':
            // Handle failed payment
            break;
    }
}
```

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-subscription-billing
- whmcs-webhook-integration
