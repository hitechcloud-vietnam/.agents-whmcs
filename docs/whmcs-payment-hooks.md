# WHMCS Payment Hooks

## Overview

Payment hooks allow customization of payment processing including authorization, capture, refunds, and payment method handling.

## Available Payment Hooks

### Payment Gateway Config

```php
<?php
// Configure payment gateway options
add_hook('GatewayConfig', 1, function(array $vars) {
    $gateway = $vars['gateway'];
    
    return [
        'test_mode' => true,
        'api_version' => 'v2',
        'webhook_url' => getWebHookUrl($gateway),
    ];
});
```

### Pre-Payment Hook

```php
<?php
// Runs before payment is processed
add_hook('PrePayment', 1, function(array $vars) {
    $gateway = $vars['gateway'];
    $amount = $vars['amount'];
    $invoiceId = $vars['invoice_id'];
    $userId = $vars['user_id'];
    
    // Validate amount
    if ($amount <= 0) {
        return [
            'abort' => true,
            'error_msg' => 'Invalid payment amount',
        ];
    }
    
    // Check fraud score
    $riskScore = calculatePaymentRisk($userId, $amount, $gateway);
    if ($riskScore > 0.7) {
        return [
            'abort' => true,
            'error_msg' => 'Payment flagged for review',
        ];
    }
    
    // Apply payment method restrictions
    if (!canUseGateway($userId, $gateway)) {
        return [
            'abort' => true,
            'error_msg' => 'Payment method not available for your account',
        ];
    }
    
    return ['abort' => false];
});

function calculatePaymentRisk(int $userId, float $amount, string $gateway): float
{
    $score = 0.0;
    
    // Check order history
    $orderCount = Capsule::table('tblorders')
        ->where('userid', $userId)
        ->whereIn('status', ['Completed', 'Active'])
        ->count();
    
    if ($orderCount === 0) {
        $score += 0.3; // New customer
    }
    
    // Check amount against average
    $avgOrder = Capsule::table('tblorders')
        ->where('userid', $userId)
        ->whereIn('status', ['Completed', 'Active'])
        ->avg('totaldue');
    
    if ($avgOrder && $amount > ($avgOrder * 3)) {
        $score += 0.3;
    }
    
    // Check for high-risk countries
    $client = Capsule::table('tblclients')->where('id', $userId)->first();
    if (in_array($client->country, getHighRiskCountries())) {
        $score += 0.2;
    }
    
    return min($score, 1.0);
}
```

### Payment Received Hook

```php
<?php
// Triggered when payment is received
add_hook('PaymentReceived', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $amount = $vars['amount'];
    $gateway = $vars['gateway'];
    $transactionId = $vars['trans_id'];
    $userId = $vars['user_id'];
    
    // Record payment details
    recordPaymentDetails($invoiceId, $vars);
    
    // Update accounting
    syncToAccounting($invoiceId, $transactionId, $amount);
    
    // Unlock services
    unlockPaidServices($invoiceId);
    
    // Send confirmation
    sendPaymentConfirmation($invoiceId);
    
    // Award loyalty points
    awardPaymentPoints($userId, $amount);
    
    // Process affiliate commission
    processAffiliateCommission($userId, $amount);
    
    return ['success' => true];
});
```

### Payment Failed Hook

```php
<?php
// Triggered when payment fails
add_hook('PaymentFailed', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $gateway = $vars['gateway'];
    $error = $vars['error'];
    $userId = $vars['user_id'];
    
    // Log failure
    logPaymentFailure($invoiceId, $gateway, $error);
    
    // Check for repeated failures
    $failCount = getPaymentFailureCount($invoiceId);
    if ($failCount >= 3) {
        // Lock payment method
        lockPaymentMethod($userId, $gateway);
        
        // Notify admin
        notifyAdminPaymentIssues($invoiceId);
    }
    
    // Send retry instructions
    sendPaymentRetryInstructions($invoiceId, $failCount);
    
    return ['success' => true];
});

function logPaymentFailure(int $invoiceId, string $gateway, string $error): void
{
    Capsule::table('mod_payment_failures')->insert([
        'invoice_id' => $invoiceId,
        'gateway' => $gateway,
        'error' => $error,
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Refund Processed Hook

```php
<?php
// Triggered when refund is processed
add_hook('RefundProcessed', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $amount = $vars['refund_amount'];
    $transactionId = $vars['original_trans_id'];
    $refundTransId = $vars['refund_trans_id'];
    $gateway = $vars['gateway'];
    
    // Update accounting
    reverseAccountingEntry($invoiceId, $transactionId, $refundTransId);
    
    // Reverse loyalty points
    reversePoints($vars['user_id'], $amount);
    
    // Reverse affiliate commission
    reverseAffiliateCommission($transactionId);
    
    // Send refund confirmation
    sendRefundConfirmation($invoiceId, $amount, $refundTransId);
    
    return ['success' => true];
});
```

### Pre-Refund Hook

```php
<?php
// Runs before refund is processed
add_hook('PreRefund', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $amount = $vars['amount'];
    $userId = $vars['user_id'];
    
    // Check if refund is within policy
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
    $daysSincePayment = (time() - strtotime($invoice->datepaid)) / 86400;
    
    if ($daysSincePayment > 30) {
        return [
            'abort' => true,
            'error_msg' => 'Refund window has expired (30 days)',
        ];
    }
    
    // Check for partial refund limit
    $existingRefunds = getExistingRefunds($invoiceId);
    $maxRefund = $invoice->total - $existingRefunds;
    
    if ($amount > $maxRefund) {
        return [
            'abort' => true,
            'error_msg' => 'Refund amount exceeds available balance',
        ];
    }
    
    // Require approval for large refunds
    if ($amount > 1000) {
        requireRefundApproval($invoiceId, $amount);
    }
    
    return ['abort' => false];
});
```

### Recurring Payment Hook

```php
<?php
// Triggered for subscription/recurring payments
add_hook('RecurringPaymentReceived', 1, function(array $vars) {
    $subscriptionId = $vars['subscription_id'];
    $amount = $vars['amount'];
    $userId = $vars['user_id'];
    
    // Process recurring payment
    processRecurringPayment($subscriptionId, $amount);
    
    // Update subscription next billing
    updateNextBillingDate($subscriptionId);
    
    // Send receipt
    sendRecurringReceipt($subscriptionId, $amount);
    
    return ['success' => true];
});
```

## Comprehensive Payment Handler

```php
<?php
class PaymentHookHandler {
    
    public function register(): void
    {
        add_hook('PrePayment', 1, [$this, 'handlePrePayment']);
        add_hook('PaymentReceived', 1, [$this, 'handleReceived']);
        add_hook('PaymentFailed', 1, [$this, 'handleFailed']);
        add_hook('RefundProcessed', 1, [$this, 'handleRefund']);
        add_hook('PreRefund', 1, [$this, 'handlePreRefund']);
        add_hook('RecurringPaymentReceived', 1, [$this, 'handleRecurring']);
    }
    
    public function handlePrePayment(array $vars): array
    {
        if (!$this->validatePayment($vars)) {
            return ['abort' => true, 'error_msg' => 'Validation failed'];
        }
        
        return ['abort' => false];
    }
    
    public function handleReceived(array $vars): array
    {
        $this->recordPayment($vars);
        $this->syncAccounting($vars);
        $this->unlockServices($vars['invoice_id']);
        $this->awardPoints($vars['user_id'], $vars['amount']);
        return ['success' => true];
    }
    
    public function handleFailed(array $vars): array
    {
        $this->logFailure($vars);
        $this->notifyFailure($vars);
        return ['success' => true];
    }
    
    public function handleRefund(array $vars): array
    {
        $this->reverseAccounting($vars);
        $this->reversePoints($vars);
        $this->sendRefundNotice($vars);
        return ['success' => true];
    }
    
    public function handlePreRefund(array $vars): array
    {
        if (!$this->validateRefund($vars)) {
            return ['abort' => true, 'error_msg' => 'Refund not allowed'];
        }
        
        return ['abort' => false];
    }
    
    public function handleRecurring(array $vars): array
    {
        $this->processRecurring($vars);
        $this->updateBillingDate($vars['subscription_id']);
        return ['success' => true];
    }
    
    private function validatePayment(array $vars): bool
    {
        return true;
    }
    
    private function recordPayment(array $vars): void
    {
        Capsule::table('mod_payment_records')->insert([
            'invoice_id' => $vars['invoice_id'],
            'gateway' => $vars['gateway'],
            'amount' => $vars['amount'],
            'trans_id' => $vars['trans_id'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function syncAccounting(array $vars): void
    {
        // Sync to accounting
    }
    
    private function unlockServices(int $invoiceId): void
    {
        // Unlock services
    }
    
    private function awardPoints(int $userId, float $amount): void
    {
        $points = (int) ($amount * 10);
        Capsule::table('mod_loyalty_points')
            ->where('user_id', $userId)
            ->increment('points', $points);
    }
    
    private function logFailure(array $vars): void
    {
        Capsule::table('mod_payment_failures')->insert([
            'invoice_id' => $vars['invoice_id'],
            'gateway' => $vars['gateway'],
            'error' => $vars['error'] ?? 'Unknown',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function notifyFailure(array $vars): void
    {
        // Notify customer
    }
    
    private function reverseAccounting(array $vars): void
    {
        // Reverse accounting
    }
    
    private function reversePoints(array $vars): void
    {
        // Reverse points
    }
    
    private function sendRefundNotice(array $vars): void
    {
        // Send refund notice
    }
    
    private function validateRefund(array $vars): bool
    {
        return true;
    }
    
    private function processRecurring(array $vars): void
    {
        // Process recurring
    }
    
    private function updateBillingDate(string $subscriptionId): void
    {
        // Update billing date
    }
}

$handler = new PaymentHookHandler();
$handler->register();
```

## Best Practices

1. **Always verify amounts** - Never trust client-side values
2. **Use transactions** - Wrap payment operations
3. **Log everything** - Maintain complete audit trail
4. **Handle idempotency** - Prevent duplicate processing
5. **Queue notifications** - Send emails asynchronously

## Related Documentation

- [WHMCS Invoice Hooks](/docs/whmcs-invoice-hooks.md)
- [WHMCS Payment Gateway Development](/docs/whmcs-gateway-dev.md)