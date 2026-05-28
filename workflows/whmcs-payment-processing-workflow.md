# WHMCS Payment Processing Workflow

## Purpose

Process payments through WHMCS payment gateways, handle transaction verification, and manage payment reconciliation for accurate financial records.

## Prerequisites

- WHMCS installation with admin access
- Payment gateway module installed and configured
- Merchant account with payment processor
- SSL certificate for secure payment processing
- Webhook/Callback URL configured with payment provider

## Workflow Steps

### Step 1: Configure Payment Gateway

Set up the primary payment gateway in WHMCS:

```php
// File: /modules/gateways/{gateway_name}.php

function {gateway}_config(): array {
    return [
        'FriendlyName' => ['value' => 'Gateway Name'],
        'merchant_id' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '50',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => 'sandbox,production',
            'Default' => 'sandbox',
        ],
    ];
}
```

Configure in WHMCS Admin:
- Navigate to Configuration > System Settings > Payment Gateways
- Activate the gateway module
- Enter API credentials from payment provider
- Set as default gateway if preferred

### Step 2: Implement Payment Link Function

Create the payment form generation:

```php
function {gateway}_link(array $params): string {
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $clientEmail = $params['clientdetails']['email'];
    
    // Generate unique transaction reference
    $transactionRef = 'INV-' . $invoiceId . '-' . time();
    
    // Store transaction reference for callback verification
    Capsule::table('mod_gateway_transactions')->insert([
        'invoice_id' => $invoiceId,
        'transaction_ref' => $transactionRef,
        'amount' => $amount,
        'status' => 'pending',
        'created_at' => Capsule::raw('NOW()')
    ]);
    
    // Build payment form
    $form = '<form action="https://api.gateway.com/checkout" method="POST">';
    $form .= '<input type="hidden" name="merchant_id" value="' . $params['merchant_id'] . '">';
    $form .= '<input type="hidden" name="amount" value="' . $amount . '">';
    $form .= '<input type="hidden" name="currency" value="' . $params['currency'] . '">';
    $form .= '<input type="hidden" name="order_id" value="' . $transactionRef . '">';
    $form .= '<input type="hidden" name="customer_email" value="' . $clientEmail . '">';
    $form .= '<input type="hidden" name="return_url" value="' . $params['systemurl'] . '/viewinvoice.php?id=' . $invoiceId . '">';
    $form .= '<input type="hidden" name="callback_url" value="' . $params['systemurl'] . '/modules/gateways/callback/' . basename(__FILE__, '.php') . '.php">';
    $form .= '<input type="submit" value="Pay with Gateway">';
    $form .= '</form>';
    
    return $form;
}
```

### Step 3: Create Payment Callback Handler

Handle payment confirmation:

```php
// File: /modules/gateways/callback/{gateway}.php

// Verify callback authenticity
function verifyWebhookSignature($payload, $signature, $secret) {
    $expectedSignature = hash_hmac('sha256', $payload, $secret);
    return hash_equals($expectedSignature, $signature);
}

// Process callback
add_hook('GatewayPaymentCallback', 1, function($vars) {
    $gatewayName = basename(__FILE__, '.php');
    
    // Get raw input for signature verification
    $rawInput = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
    
    // Verify signature (adapt to your gateway)
    $apiKey = Capsule::table('tblpaymentgateways')
        ->where('gateway', $gatewayName)
        ->where('setting', 'api_key')
        ->value('value');
    
    if (!verifyWebhookSignature($rawInput, $signature, $apiKey)) {
        logTransaction($gatewayName, $_POST, 'Verification Failed');
        http_response_code(403);
        exit('Invalid signature');
    }
    
    // Extract payment data
    $transactionRef = $_POST['order_id'] ?? '';
    $status = $_POST['status'] ?? '';
    $amount = $_POST['amount'] ?? 0;
    
    // Find the pending transaction
    $transaction = Capsule::table('mod_gateway_transactions')
        ->where('transaction_ref', $transactionRef)
        ->where('status', 'pending')
        ->first();
    
    if (!$transaction) {
        logTransaction($gatewayName, $_POST, 'Transaction Not Found');
        exit('Transaction not found');
    }
    
    // Process based on payment status
    if ($status === 'completed') {
        // Add payment to WHMCS invoice
        addInvoicePayment(
            $transaction->invoice_id,
            $_POST['transaction_id'] ?? '',
            $amount,
            0,
            $gatewayName
        );
        
        // Update transaction record
        Capsule::table('mod_gateway_transactions')
            ->where('id', $transaction->id)
            ->update([
                'status' => 'completed',
                'completed_at' => Capsule::raw('NOW()'),
                'gateway_transaction_id' => $_POST['transaction_id'] ?? ''
            ]);
        
        logTransaction($gatewayName, $_POST, 'Success');
    } elseif ($status === 'failed') {
        Capsule::table('mod_gateway_transactions')
            ->where('id', $transaction->id)
            ->update(['status' => 'failed']);
        
        logTransaction($gatewayName, $_POST, 'Failed');
    }
    
    echo 'OK';
});
```

### Step 4: Handle Payment Errors

Implement error handling:

```php
// File: /includes/hooks/payment_error_handler.php

add_hook('InvoicePaymentFailed', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $clientId = $vars['userid'];
    
    // Get client details
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    // Log the failure
    logActivity("Payment failed for invoice #{$invoiceId}. Client: {$client->email}");
    
    // Check if this is a recurring failure
    $recentFailures = Capsule::table('tblgatewaylog')
        ->where('userid', $clientId)
        ->whereDate('date', '>=', date('Y-m-d', strtotime('-7 days')))
        ->count();
    
    if ($recentFailures >= 3) {
        // Notify admin of potential issue
        sendAdminNotification(
            'email',
            'Payment Issue Alert',
            "Client {$client->firstname} {$client->lastname} ({$client->email}) has failed {$recentFailures} payments in the past week."
        );
    }
});

add_hook('DailyCronJob', 1, function($vars) {
    // Check for stuck pending transactions (older than 30 minutes)
    $stuckTransactions = Capsule::table('mod_gateway_transactions')
        ->where('status', 'pending')
        ->where('created_at', '<', Capsule::raw('NOW() - INTERVAL 30 MINUTE'))
        ->get();
    
    foreach ($stuckTransactions as $transaction) {
        // Mark as expired
        Capsule::table('mod_gateway_transactions')
            ->where('id', $transaction->id)
            ->update(['status' => 'expired']);
        
        // Notify client to retry payment
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $transaction->invoice_id)
            ->first();
        
        sendTemplatedEmail('Payment Expired', $invoice->userid, [
            'invoice_id' => $transaction->invoice_id
        ]);
    }
});
```

### Step 5: Implement Payment Reconciliation

Daily reconciliation process:

```php
// File: /includes/hooks/payment_reconciliation.php

add_hook('DailyCronJob', 1, function($vars) {
    $yesterday = date('Y-m-d', strtotime('-1 day'));
    
    // Get all payments from yesterday
    $payments = Capsule::table('tblinvoices')
        ->where('status', 'Paid')
        ->whereDate('datepaid', $yesterday)
        ->get();
    
    $reconciliationReport = [
        'date' => $yesterday,
        'total_transactions' => count($payments),
        'total_amount' => 0,
        'by_gateway' => [],
        'discrepancies' => []
    ];
    
    foreach ($payments as $payment) {
        $reconciliationReport['total_amount'] += $payment->total;
        
        $gateway = $payment->paymentmethod ?? 'unknown';
        if (!isset($reconciliationReport['by_gateway'][$gateway])) {
            $reconciliationReport['by_gateway'][$gateway] = ['count' => 0, 'amount' => 0];
        }
        $reconciliationReport['by_gateway'][$gateway]['count']++;
        $reconciliationReport['by_gateway'][$gateway]['amount'] += $payment->total;
        
        // Verify with gateway's records
        $transaction = Capsule::table('mod_gateway_transactions')
            ->where('invoice_id', $payment->id)
            ->where('status', 'completed')
            ->first();
        
        if (!$transaction) {
            $reconciliationReport['discrepancies'][] = [
                'invoice_id' => $payment->id,
                'issue' => 'No matching gateway transaction found'
            ];
        }
    }
    
    // Save reconciliation report
    $reportJson = json_encode($reconciliationReport, JSON_PRETTY_PRINT);
    file_put_contents(
        __DIR__ . "/../storage/logs/reconciliation/{$yesterday}.json",
        $reportJson
    );
    
    // Alert on discrepancies
    if (count($reconciliationReport['discrepancies']) > 0) {
        sendAdminNotification(
            'email',
            'Payment Reconciliation Alert',
            "Found " . count($reconciliationReport['discrepancies']) . " discrepancies on {$yesterday}. See attached report."
        );
    }
});
```

## Verification Checklist

- [ ] Payment gateway module installed and activated
- [ ] API credentials configured correctly
- [ ] Webhook/Callback URL set in payment provider dashboard
- [ ] Test transaction completed successfully
- [ ] Payment reflected in WHMCS admin panel
- [ ] Email confirmation sent to client
- [ ] Transaction logged in WHMCS gateway log
- [ ] Refund functionality tested (if applicable)
- [ ] Error handling verified with failed payment scenarios
- [ ] Daily reconciliation reports generated correctly

## Related Skills and Documentation

- [WHMCS Payment Gateway Setup](whmcs-payment-gateway-setup-workflow.md)
- [WHMCS Payment Reconciliation](whmcs-payment-reconciliation-workflow.md)
- [WHMCS Refund Workflow](whmcs-refund-workflow.md)
- WHMCS Documentation: Payment Gateway Development
- WHMCS Documentation: Transaction Logging

## Notes

- Always use SSL for payment processing
- Store payment credentials securely
- Implement proper webhook signature verification
- Monitor for failed transactions and address quickly
- Keep gateway module updated for security patches
- Test in sandbox mode before production deployment