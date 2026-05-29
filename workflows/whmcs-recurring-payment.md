# WHMCS Recurring Payment Workflow

## Description
Configure and manage recurring payments in WHMCS.

## Steps

### Step 1: Configure Recurring Pricing
```bash
# In WHMCS Admin:
# Products > Edit Product > Pricing
# Set up multiple billing cycles:
# - Monthly
# - Quarterly
# - Semi-Annual
# - Annual
# - Biennial
# - Triennial
```

### Step 2: Payment Gateway Setup
```php
<?php
// Ensure gateway supports recurring

function clicodes_gateway_config()
{
    return [
        // ... basic config
        'supportsRecurringBilling' => true,
        'recurringBillingNote' => 'Recurring payments are processed automatically',
    ];
}
```

### Step 3: Configure Auto Registration
```bash
# Configuration > System Settings > Automation Settings
# Ensure "Create Invoices" and "Recurring Invoices" are enabled

# Cron should run:
# */5 * * * * php -q /path/to/crons/cron.php
```

### Step 4: Handle Failed Recurring Payments
```php
<?php
// Hook for failed invoice payment

add_hook('InvoicePaymentFailed', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    
    // Send reminder
    sendEmail('InvoicePaymentFailed', $vars['userid']);
    
    // Add to retry queue
    Capsule::table('mod_payment_retries')->insert([
        'invoice_id' => $invoiceId,
        'attempts' => 0,
        'next_retry' => date('Y-m-d', strtotime('+3 days')),
    ]);
    
    // Suspend service after X failed attempts
    $failedCount = getFailedPaymentCount($invoiceId);
    if ($failedCount >= 3) {
        suspendService($vars['service_id']);
    }
});
```

### Step 5: Retry Failed Payments
```php
<?php
// Cron job for retry

add_hook('DailyCronJob', 1, function() {
    $pendingRetries = Capsule::table('mod_payment_retries')
        ->where('next_retry', '<=', date('Y-m-d H:i:s'))
        ->where('attempts', '<', 5)
        ->get();
    
    foreach ($pendingRetries as $retry) {
        retryPayment($retry->invoice_id);
        
        Capsule::table('mod_payment_retries')
            ->where('id', $retry->id)
            ->update([
                'attempts' => $retry->attempts + 1,
                'next_retry' => date('Y-m-d', strtotime('+' . (3 * ($retry->attempts + 1)) . ' days')),
            ]);
    }
});
```

## Recurring Payment Timeline
```
Day 0: Invoice created
Day 1: Payment due reminder
Day 7: First attempt + fail notification
Day 10: Retry attempt 1
Day 13: Retry attempt 2
Day 16: Retry attempt 3
Day 17: Service suspended
Day 30: Account cancellation (if still unpaid)
```

## Tags
- recurring
- payment
- automation
- billing