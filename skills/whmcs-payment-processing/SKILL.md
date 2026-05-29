# WHMCS Payment Processing

## Overview
Master skill for payment processing in WHMCS. Covers payment gateways, transaction management, webhook handling, and payment automation.

## Payment Processing Hooks

```php
<?php
// /includes/hooks/payment_hooks.php

/**
 * Before payment processing
 */
add_hook('PrePaymentProcessing', 1, function(array $params) {
    // Validate payment amount
    if ($params['amount'] <= 0) {
        return [
            'success' => false,
            'error' => 'Invalid payment amount',
        ];
    }

    // Check for duplicate transactions
    $existing = \Illuminate\Database\Capsule\Manager::table('tblaccounts')
        ->where('transid', $params['transid'] ?? '')
        ->first();

    if ($existing) {
        return [
            'success' => false,
            'error' => 'Duplicate transaction detected',
        ];
    }

    // Fraud screening
    $riskScore = calculatePaymentRisk($params);

    if ($riskScore > 80) {
        return [
            'success' => false,
            'error' => 'Payment flagged for review',
        ];
    }

    return ['success' => true];
});

/**
 * After successful payment
 */
add_hook('AfterPaymentProcessing', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];

    // Update CRM
    syncPaymentToCRM($invoiceId, $amount);

    // Process affiliate commission
    processAffiliateCommission($params['userid'], $amount);

    // Update loyalty points
    updateLoyaltyPoints($params['userid'], $amount);

    // Send confirmation email
    send_email('PaymentConfirmation', $params['userid'], [
        'invoice_id' => $invoiceId,
        'amount' => $amount,
        'transaction_id' => $params['transid'],
    ]);

    return true;
});

/**
 * Handle payment webhook
 */
add_hook('PaymentWebhook', 1, function(array $params) {
    $gateway = $params['gateway'];
    $event = $params['event'];
    $data = $params['data'];

    switch ($event) {
        case 'payment.success':
            handleSuccessfulPayment($data);
            break;

        case 'payment.failed':
            handleFailedPayment($data);
            break;

        case 'refund.processed':
            handleRefundProcessed($data);
            break;

        case 'dispute.opened':
            handleDisputeOpened($data);
            break;

        case 'dispute.resolved':
            handleDisputeResolved($data);
            break;
    }

    return true;
});

/**
 * Payment method validation
 */
add_hook('ValidatePaymentMethod', 1, function(array $params) {
    $errors = [];

    // Validate credit card
    if ($params['payment_method'] === 'credit_card') {
        if (!validateCreditCard($params['card_number'])) {
            $errors[] = 'Invalid credit card number';
        }

        if (!validateCardExpiry($params['card_expiry'])) {
            $errors[] = 'Card has expired';
        }

        if (!validateCardCVC($params['card_cvc'])) {
            $errors[] = 'Invalid CVC';
        }
    }

    // Validate bank account
    if ($params['payment_method'] === 'bank_transfer') {
        if (empty($params['bank_account'])) {
            $errors[] = 'Bank account required';
        }
    }

    if (!empty($errors)) {
        return [
            'success' => false,
            'errors' => $errors,
        ];
    }

    return ['success' => true];
});
```

## Payment Helper Functions

```php
<?php
// /includes/helpers/payment_helper.php

/**
 * Process payment for an invoice
 */
function processInvoicePayment(int $invoiceId, float $amount, string $paymentMethod = ''): array
{
    $invoice = \WHMCS\Billing\Invoice::find($invoiceId);

    if (!$invoice) {
        return [
            'success' => false,
            'error' => 'Invoice not found',
        ];
    }

    if ($invoice->status === 'Paid') {
        return [
            'success' => false,
            'error' => 'Invoice already paid',
        ];
    }

    // Apply payment
    $result = localapi('AddInvoicePayment', [
        'invoiceid' => $invoiceId,
        'transid' => generateTransactionId(),
        'amount' => $amount,
        'gateway' => $paymentMethod,
    ]);

    return $result;
}

/**
 * Validate credit card number (Luhn algorithm)
 */
function validateCreditCard(string $number): bool
{
    $number = preg_replace('/\D/', '', $number);

    if (strlen($number) < 13 || strlen($number) > 19) {
        return false;
    }

    $sum = 0;
    $isAlternate = false;

    for ($i = strlen($number) - 1; $i >= 0; $i--) {
        $digit = (int)$number[$i];

        if ($isAlternate) {
            $digit *= 2;
            if ($digit > 9) {
                $digit -= 9;
            }
        }

        $sum += $digit;
        $isAlternate = !$isAlternate;
    }

    return ($sum % 10 === 0);
}

/**
 * Get card type from number
 */
function getCardType(string $number): string
{
    $number = preg_replace('/\D/', '', $number);

    $patterns = [
        'visa' => '/^4/',
        'mastercard' => '/^5[1-5]/',
        'amex' => '/^3[47]/',
        'discover' => '/^6(?:011|5)/',
        'diners' => '/^3(?:0[0-5]|[68])/',
        'jcb' => '/^35/',
    ];

    foreach ($patterns as $type => $pattern) {
        if (preg_match($pattern, $number)) {
            return $type;
        }
    }

    return 'unknown';
}

/**
 * Validate card expiry date
 */
function validateCardExpiry(string $expiry): bool
{
    if (!preg_match('/^(0[1-9]|1[0-2])\/([0-9]{2})$/', $expiry, $matches)) {
        return false;
    }

    $month = (int)$matches[1];
    $year = 2000 + (int)$matches[2];

    $expiryDate = new \DateTime("{$year}-{$month}-01");
    $expiryDate->modify('last day of this month');
    $expiryDate->setTime(23, 59, 59);

    return $expiryDate >= new \DateTime();
}

/**
 * Calculate payment risk score
 */
function calculatePaymentRisk(array $params): int
{
    $score = 0;

    // Check IP reputation
    if (isHighRiskIP($params['ip'])) {
        $score += 30;
    }

    // Check email domain
    $emailDomain = explode('@', $params['email'])[1] ?? '';
    if (isFreeEmailProvider($emailDomain)) {
        $score += 10;
    }

    // Check country
    if (isHighRiskCountry($params['country'])) {
        $score += 20;
    }

    // Check amount
    if ($params['amount'] > 1000) {
        $score += 15;
    }

    // Check for previous failed payments
    if (hasFailedPayments($params['userid'])) {
        $score += 25;
    }

    return min($score, 100);
}

/**
 * Generate unique transaction ID
 */
function generateTransactionId(): string
{
    return strtoupper(bin2hex(random_bytes(8)));
}

/**
 * Get payment summary for client
 */
function getPaymentSummary(int $clientId): array
{
    $client = \WHMCS\User\Client::find($clientId);

    return [
        'total_paid' => $client->invoices()->where('status', 'Paid')->sum('total'),
        'total_outstanding' => $client->invoices()->where('status', 'Unpaid')->sum('total'),
        'total_overdue' => $client->invoices()->where('status', 'Unpaid')
            ->where('duedate', '<', date('Y-m-d'))
            ->sum('total'),
        'credit_balance' => $client->credit,
        'pending_invoices' => $client->invoices()->where('status', 'Unpaid')->count(),
    ];
}
```

## Payment Gateway Integration

```php
<?php
// /includes/integrations/payment_processor.php

class PaymentProcessor
{
    private $gateways = [];

    public function __construct()
    {
        $this->loadActiveGateways();
    }

    private function loadActiveGateways(): void
    {
        $this->gateways = \Illuminate\Database\Capsule\Manager::table('tblpaymentgateways')
            ->where('setting', 'name')
            ->where('value', '!=', '')
            ->get();
    }

    public function processPayment(int $invoiceId, string $gateway, array $paymentData): array
    {
        $invoice = \WHMCS\Billing\Invoice::find($invoiceId);

        if (!$invoice) {
            return [
                'success' => false,
                'error' => 'Invoice not found',
            ];
        }

        // Get gateway settings
        $gatewaySettings = $this->getGatewaySettings($gateway);

        // Create payment intent
        $paymentIntent = $this->createPaymentIntent($gatewaySettings, [
            'amount' => $invoice->total * 100, // Convert to cents
            'currency' => strtolower($invoice->currency->code),
            'email' => $invoice->client->email,
            'metadata' => [
                'invoice_id' => $invoiceId,
                'client_id' => $invoice->clientId,
            ],
        ]);

        if (!$paymentIntent['success']) {
            return $paymentIntent;
        }

        // Confirm payment
        $result = $this->confirmPayment($gatewaySettings, $paymentIntent['id'], $paymentData);

        if ($result['success']) {
            // Apply payment to invoice
            $this->applyPayment($invoice, $result);
        }

        return $result;
    }

    private function getGatewaySettings(string $gateway): array
    {
        return \Illuminate\Database\Capsule\Manager::table('tblpaymentgateways')
            ->where('gateway', $gateway)
            ->pluck('value', 'setting')
            ->toArray();
    }

    private function createPaymentIntent(array $settings, array $data): array
    {
        $api = new PaymentGatewayAPI($settings);

        try {
            return $api->createPaymentIntent($data);
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    private function confirmPayment(array $settings, string $intentId, array $data): array
    {
        $api = new PaymentGatewayAPI($settings);

        try {
            return $api->confirmPayment($intentId, $data);
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    private function applyPayment(\WHMCS\Billing\Invoice $invoice, array $paymentResult): void
    {
        addInvoicePayment(
            $invoice->id,
            $paymentResult['transaction_id'],
            $paymentResult['amount'] / 100,
            0,
            $paymentResult['gateway'] ?? 'credit_card'
        );
    }
}
```

## Best Practices

1. **Always verify payments**: Validate all payment data before processing
2. **Use webhooks**: Implement webhook handlers for real-time updates
3. **Idempotency**: Handle duplicate webhook events gracefully
4. **PCI compliance**: Never store full credit card numbers
5. **Logging**: Log all payment attempts and results
6. **Error handling**: Provide clear error messages to users
7. **Fraud prevention**: Implement fraud detection mechanisms
8. **Timeout handling**: Set appropriate timeouts for payment requests
9. **Retry logic**: Implement retry logic with exponential backoff
10. **Testing**: Test all payment scenarios including edge cases
