# WHMCS ARB Completion Workflow

## Overview
This workflow automates ACH/bank verification and recurring billing.

## Prerequisites
- WHMCS installation
- Bank payment gateway

## Step-by-Step Guide

### Step 1: Configure ARB
```php
// lib/ARBService.php
class ARBService
{
    public function create_subscription(array $params): array
    {
        return [
            'subscription_id' => $this->createARB($params),
            'status' => 'pending',
        ];
    }
    
    public function verify_bank_account(string $subscriptionId, array $microDeposits): bool
    {
        // Verify micro deposits
        $result = $this->validateMicroDeposits($subscriptionId, $microDeposits);
        
        if ($result['verified']) {
            $this->updateSubscriptionStatus($subscriptionId, 'active');
            return true;
        }
        
        return false;
    }
}
```

### Step 2: Create Verification Hook
```php
add_hook('InvoiceCreated', 1, function($vars) {
    $invoiceId = $vars['invoiceId'];
    $invoice = \WHMCS\Billing\Invoice::find($invoiceId);
    
    // Check if bank payment
    if ($invoice->paymentmethod !== 'bank_transfer') {
        return;
    }
    
    $client = $invoice->client;
    $subscription = get_bank_subscription($client->id);
    
    if ($subscription && $subscription->status === 'pending') {
        // Invoice will be paid via ARB
        log_arb_invoice($invoiceId, $subscription->id);
    }
});

add_hook('AfterCronJob', 1, function() {
    // Process pending ARB payments
    $pendingInvoices = get_pending_arb_invoices();
    
    foreach ($pendingInvoices as $invoice) {
        $result = process_arb_payment($invoice);
        
        if (!$result['success']) {
            handle_arb_failure($invoice->id, $result['error']);
        }
    }
});
```

## ARB Completion Checklist

### Setup
- [ ] Bank gateway configured
- [ ] ARB subscription created
- [ ] Micro deposits verified

### Processing
- [ ] Invoices generated
- [ ] Payments processed
- [ ] Failures handled

### Completion
- [ ] Payment confirmed
- [ ] Subscription active
- [ ] Records updated
