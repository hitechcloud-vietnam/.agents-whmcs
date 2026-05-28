# WHMCS Refund Workflow

## Purpose

Process customer refunds through WHMCS, handle partial and full refunds, and maintain accurate financial records for audit compliance.

## Prerequisites

- WHMCS admin access with billing permissions
- Payment gateway with refund capability
- Original transaction details
- Customer refund request documentation
- Refund policy established and documented

## Workflow Steps

### Step 1: Review Refund Request

Validate the refund request before processing:

```php
// Check original transaction details
$invoiceId = (int) $_POST['invoice_id'];
$refundAmount = (float) $_POST['refund_amount'];
$reason = sanitize($_POST['refund_reason']);

$invoice = Capsule::table('tblinvoices')
    ->where('id', $invoiceId)
    ->first();

$transaction = Capsule::table('tblaccounts')
    ->where('invoiceid', $invoiceId)
    ->where('transid', '!=', '')
    ->orderBy('id', 'desc')
    ->first();

if (!$transaction) {
    throw new Exception('Original payment transaction not found');
}

// Validate refund amount
$maxRefundable = $transaction->amount;
if ($refundAmount > $maxRefundable) {
    throw new Exception("Refund amount exceeds original payment of {$maxRefundable}");
}
```

### Step 2: Process Refund via Payment Gateway

Initiate refund through the original payment gateway:

```php
// File: /includes/hooks/refund_processing.php

function processRefund($transactionId, $amount, $gateway, $invoiceId) {
    // Get gateway module configuration
    $gatewayConfig = Capsule::table('tblpaymentgateways')
        ->where('gateway', $gateway)
        ->pluck('value', 'setting')
        ->toArray();
    
    switch ($gateway) {
        case 'stripe':
            return processStripeRefund($transactionId, $amount, $gatewayConfig);
        case 'paypal':
            return processPayPalRefund($transactionId, $amount, $gatewayConfig);
        case 'custom_gateway':
            return processCustomGatewayRefund($transactionId, $amount, $gatewayConfig);
        default:
            throw new Exception("Gateway {$gateway} does not support refunds");
    }
}

function processStripeRefund($transactionId, $amount, $config) {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://api.stripe.com/v1/refunds',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query([
            'charge' => $transactionId,
            'amount' => (int)($amount * 100), // Stripe uses cents
        ]),
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $config['api_key']
        ]
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if ($httpCode !== 200) {
        logActivity("Stripe refund failed: " . ($result['error']['message'] ?? 'Unknown error'));
        return ['success' => false, 'error' => $result['error']['message'] ?? 'Refund failed'];
    }
    
    return [
        'success' => true,
        'refund_id' => $result['id'],
        'status' => $result['status']
    ];
}

function processPayPalRefund($transactionId, $amount, $config) {
    // PayPal refund requires capturing first, then refunding
    // This is a simplified example - adapt to your PayPal integration
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://api.paypal.com/v2/payments/captures/' . $transactionId . '/refund',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'amount' => [
                'value' => number_format($amount, 2, '.', ''),
                'currency_code' => 'USD'
            ]
        ]),
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $config['access_token'],
            'Content-Type: application/json'
        ]
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($response, true);
}
```

### Step 3: Record Refund in WHMCS

Create refund transaction record:

```php
// After successful gateway refund
function recordRefundInWHMCS($invoiceId, $refundAmount, $gateway, $refundId, $originalTransId) {
    // Create negative transaction entry
    Capsule::table('tblaccounts')->insert([
        'userid' => $invoice->userid,
        'invoiceid' => $invoiceId,
        'description' => "Refund for transaction {$originalTransId}",
        'amount' => -$refundAmount,
        'date' => date('Y-m-d H:i:s'),
        'transid' => $refundId,
        'gateway' => $gateway
    ]);
    
    // Update invoice balance (optional - depends on your billing approach)
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
    $newBalance = $invoice->balance + $refundAmount;
    
    Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->update(['balance' => $newBalance]);
    
    // Log the refund
    logActivity("Refund processed for invoice #{$invoiceId}: {$refundAmount} via {$gateway}");
    logTransaction($gateway, [
        'type' => 'Refund',
        'original_transaction' => $originalTransId,
        'refund_transaction' => $refundId,
        'amount' => $refundAmount
    ], 'Refunded');
    
    // Update tracking table
    Capsule::table('mod_refund_tracking')->insert([
        'invoice_id' => $invoiceId,
        'original_transaction_id' => $originalTransId,
        'refund_transaction_id' => $refundId,
        'amount' => $refundAmount,
        'reason' => $_POST['refund_reason'] ?? 'Not specified',
        'processed_by' => $_SESSION['adminid'],
        'processed_at' => Capsule::raw('NOW()')
    ]);
}
```

### Step 4: Create Refund Request Hook

Automate refund request processing:

```php
// File: /includes/hooks/refund_automation.php

add_hook('DailyCronJob', 1, function($vars) {
    // Check for pending refund requests
    $pendingRequests = Capsule::table('mod_refund_requests')
        ->where('status', 'pending')
        ->where('created_at', '<', Capsule::raw('NOW() - INTERVAL 48 HOUR'))
        ->get();
    
    foreach ($pendingRequests as $request) {
        // Send reminder to billing team
        $admin = Capsule::table('tbladmins')
            ->where('roleid', 1) // Admin role
            ->first();
        
        sendTemplatedEmail('RefundRequestPending', $admin->email, [
            'request_id' => $request->id,
            'client_name' => $request->client_name,
            'amount' => $request->amount,
            'created_at' => $request->created_at
        ]);
    }
    
    // Auto-process small refunds (under $10)
    $autoProcessable = Capsule::table('mod_refund_requests')
        ->where('status', 'pending')
        ->where('amount', '<=', 10)
        ->get();
    
    foreach ($autoProcessable as $request) {
        try {
            $result = processRefund(
                $request->original_transaction_id,
                $request->amount,
                $request->gateway,
                $request->invoice_id
            );
            
            if ($result['success']) {
                Capsule::table('mod_refund_requests')
                    ->where('id', $request->id)
                    ->update(['status' => 'completed', 'refund_id' => $result['refund_id']]);
                
                recordRefundInWHMCS(
                    $request->invoice_id,
                    $request->amount,
                    $request->gateway,
                    $result['refund_id'],
                    $request->original_transaction_id
                );
            }
        } catch (Exception $e) {
            Capsule::table('mod_refund_requests')
                ->where('id', $request->id)
                ->update(['status' => 'failed', 'error_message' => $e->getMessage()]);
        }
    }
});

add_hook('InvoiceRefunded', 1, function($vars) {
    // Send notification to client
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->first();
    
    sendTemplatedEmail('RefundProcessed', $invoice->userid, [
        'refund_amount' => $vars['amount'],
        'refund_method' => $vars['paymentmethod'],
        'transaction_id' => $vars['transactionid']
    ]);
});
```

### Step 5: Refund Reporting

Generate refund analytics:

```php
// File: /includes/hooks/refund_reporting.php

add_hook('AdminAreaPageCallback', 1, function($vars) {
    // Add refund stats to admin dashboard
    if ($vars['filename'] === 'index') {
        $refundStats = getRefundStatistics();
        
        return [
            'refund_stats' => $refundStats
        ];
    }
});

function getRefundStatistics() {
    $thisMonth = date('Y-m-01');
    $lastMonth = date('Y-m-01', strtotime('-1 month'));
    
    return [
        'this_month' => [
            'count' => Capsule::table('mod_refund_tracking')
                ->where('processed_at', '>=', $thisMonth)
                ->count(),
            'total' => Capsule::table('mod_refund_tracking')
                ->where('processed_at', '>=', $thisMonth)
                ->sum('amount')
        ],
        'last_month' => [
            'count' => Capsule::table('mod_refund_tracking')
                ->whereBetween('processed_at', [$lastMonth, $thisMonth])
                ->count(),
            'total' => Capsule::table('mod_refund_tracking')
                ->whereBetween('processed_at', [$lastMonth, $thisMonth])
                ->sum('amount')
        ],
        'pending' => Capsule::table('mod_refund_requests')
            ->where('status', 'pending')
            ->count()
    ];
}
```

## Verification Checklist

- [ ] Refund request validated against original payment
- [ ] Payment gateway refund API tested
- [ ] Refund recorded in WHMCS accounts table
- [ ] Invoice balance updated correctly
- [ ] Refund confirmation email sent to client
- [ ] Refund tracking database updated
- [ ] Admin notification sent for large refunds
- [ ] Refund appears in WHMCS transaction reports
- [ ] Reconciliation verified with payment processor
- [ ] Audit trail complete and accessible

## Related Skills and Documentation

- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- [WHMCS Payment Reconciliation](whmcs-payment-reconciliation-workflow.md)
- [WHMCS Billing Audit](whmcs-billing-audit-workflow.md)
- WHMCS Documentation: Transaction Management
- WHMCS Documentation: Payment Gateway Refunds

## Notes

- Always verify refund eligibility before processing
- Document all refund reasons for audit compliance
- Set approval thresholds for large refunds
- Monitor refund rate to detect issues early
- Keep refund records for the required retention period
- Consider partial refund vs full refund implications
- Update client account notes with refund history