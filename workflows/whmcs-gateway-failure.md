# WHMCS Payment Gateway Failure Handling Workflow

## Description
Handle and recover from payment gateway failures gracefully.

## Steps

### Step 1: Implement Failure Handling
```php
<?php
/**
 * Payment Gateway Failure Handler
 */

class PaymentFailureHandler
{
    /**
     * Handle payment failure
     */
    public function handleFailure($gateway, $error, $params)
    {
        // Log the failure
        $this->logFailure($gateway, $error, $params);
        
        // Increment failure counter
        $this->incrementFailureCounter($gateway);
        
        // Determine next action
        $action = $this->determineAction($gateway, $error);
        
        // Execute recovery action
        return $this->executeAction($action, $gateway, $params);
    }
    
    private function logFailure($gateway, $error, $params)
    {
        Capsule::table('mod_payment_failures')->insert([
            'gateway' => $gateway,
            'error_code' => $error['code'] ?? 'unknown',
            'error_message' => $error['message'] ?? json_encode($error),
            'invoice_id' => $params['invoiceid'] ?? null,
            'user_id' => $params['clientdetails']['id'] ?? null,
            'amount' => $params['amount'] ?? 0,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        logTransaction($gateway, $params, 'Failed: ' . ($error['message'] ?? 'Unknown error'));
    }
    
    private function determineAction($gateway, $error)
    {
        $failureCount = $this->getFailureCount($gateway);
        
        // Too many failures from this gateway
        if ($failureCount >= 10) {
            return 'switch_gateway';
        }
        
        // Retryable error codes
        $retryableCodes = ['timeout', 'connection_error', 'service_unavailable'];
        
        if (in_array($error['code'] ?? '', $retryableCodes)) {
            return 'retry';
        }
        
        // Input error - don't retry
        if (in_array($error['code'], ['invalid_card', 'insufficient_funds'])) {
            return 'show_error';
        }
        
        return 'retry_with_backoff';
    }
    
    private function executeAction($action, $gateway, $params)
    {
        switch ($action) {
            case 'retry':
                return $this->retryPayment($gateway, $params);
                
            case 'retry_with_backoff':
                return $this->scheduleRetry($gateway, $params);
                
            case 'switch_gateway':
                return $this->switchToBackupGateway($params);
                
            case 'show_error':
                return [
                    'result' => 'error',
                    'message' => $this->getUserFriendlyError($error['code'] ?? ''),
                    'can_retry' => true,
                ];
        }
    }
    
    private function retryPayment($gateway, $params)
    {
        // Simple retry after short delay
        sleep(2);
        return call_user_func($gateway . '_link', $params);
    }
    
    private function scheduleRetry($gateway, $params)
    {
        // Schedule retry with exponential backoff
        $invoiceId = $params['invoiceid'];
        
        Capsule::table('mod_payment_retries')->insert([
            'invoice_id' => $invoiceId,
            'gateway' => $gateway,
            'params' => json_encode($params),
            'attempt' => 1,
            'next_retry' => date('Y-m-d H:i:s', strtotime('+1 hour')),
            'status' => 'scheduled',
        ]);
        
        return [
            'result' => 'scheduled',
            'message' => 'Payment will be retried automatically within 1 hour.',
        ];
    }
    
    private function switchToBackupGateway($params)
    {
        // Get backup gateway
        $backup = Capsule::table('tblpaymentgateways')
            ->where('setting', 'is_backup')
            ->where('value', '1')
            ->first();
        
        if ($backup) {
            $gatewayParams = getGatewayVariables($backup->gateway);
            return call_user_func($backup->gateway . '_link', array_merge($params, $gatewayParams));
        }
        
        return [
            'result' => 'error',
            'message' => 'Payment processing is temporarily unavailable. Please try again later.',
        ];
    }
    
    private function getUserFriendlyError($code)
    {
        $errors = [
            'invalid_card' => 'The card details entered are invalid. Please check and try again.',
            'expired_card' => 'This card has expired. Please use a different payment method.',
            'insufficient_funds' => 'Insufficient funds in your account.',
            'card_declined' => 'Your card was declined. Please contact your bank.',
            'processing_error' => 'An error occurred processing your card. Please try again.',
            'timeout' => 'The payment processor is taking too long. Please wait.',
        ];
        
        return $errors[$code] ?? 'An error occurred processing your payment. Please try again.';
    }
}
```

### Step 2: Configure Failover
```php
<?php
// configuration.php

$payment_gateways = [
    'primary' => 'stripe',
    'failover' => 'paypal',
    'fallback' => 'bank_transfer',
];

$gateway_health = [
    'stripe' => ['status' => 'healthy', 'failures' => 0],
    'paypal' => ['status' => 'healthy', 'failures' => 0],
];
```

### Step 3: Health Check
```php
<?php
add_hook('DailyCronJob', 1, function() {
    // Check gateway health
    foreach ($gateway_health as $gateway => $health) {
        $failures = Capsule::table('mod_payment_failures')
            ->where('gateway', $gateway)
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->count();
        
        if ($failures > 10) {
            // Alert and potentially disable
            sendAlert("Gateway $gateway has high failure rate: $failures in 24h");
        }
    }
});
```

## Failure Recovery Flow
```
1. Payment attempt fails
2. Log failure with error code
3. Check failure count for gateway
4. If retryable error:
   - Schedule retry with backoff
5. If too many failures:
   - Switch to backup gateway
6. If input error:
   - Show user-friendly message
7. Process retry via cron
8. Update metrics and alerts
```

## Tags
- payment-failure
- error-handling
- failover
- recovery