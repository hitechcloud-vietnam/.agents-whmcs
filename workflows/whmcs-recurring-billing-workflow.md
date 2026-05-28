# WHMCS Recurring Billing Workflow

## Purpose

Manage recurring billing cycles, automate payment collection, handle failed recurring payments, and maintain subscription revenue consistency.

## Prerequisites

- WHMCS with recurring billing products
- Payment gateway with recurring support
- Cron job configured for daily automation
- Clear billing cycle configuration

## Workflow Steps

### Step 1: Configure Recurring Billing Settings

Set up recurring billing configuration:

```php
// Database configuration for recurring billing
INSERT INTO tblconfiguration (setting, value) VALUES 
('RecurringBillingEnabled', 'on'),
('RecurringBillingDaysAdvance', '3'),
('FailedPaymentRetryDays', '3,7,14'),
('MaxPaymentRetries', '3'),
('FailedPaymentSuspendDays', '14'),
('RecurringInvoiceEmailTemplate', 'Recurring Invoice Created');

// Create recurring billing tracking tables
Capsule::schema()->create('mod_recurring_billing', function($t) {
    $t->increments('id');
    $t->integer('hosting_id');
    $t->integer('client_id');
    $t->string('billing_cycle');
    $t->decimal('amount', 10, 2);
    $t->date('next_billing_date');
    $t->integer('retry_count')->default(0);
    $t->date('last_billing_date')->nullable();
    $t->string('status'); // active, suspended, cancelled
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});

Capsule::schema()->create('mod_recurring_payment_log', function($t) {
    $t->increments('id');
    $t->integer('recurring_id');
    $t->integer('invoice_id');
    $t->string('status'); // success, failed, pending
    $t->string('gateway_response');
    $t->timestamp('attempted_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Recurring Billing Manager

Build recurring billing functionality:

```php
// File: /includes/classes/RecurringBillingManager.php

namespace WHMCS\Billing;

class RecurringBillingManager {
    
    public function initializeRecurringBilling($hostingId) {
        $hosting = Capsule::table('tblhosting')
            ->where('id', $hostingId)
            ->first();
        
        $product = Capsule::table('tblproducts')
            ->where('id', $hosting->packageid)
            ->first();
        
        $nextBillingDate = $this->calculateNextBillingDate(
            $hosting->regdate,
            $hosting->billingcycle
        );
        
        Capsule::table('mod_recurring_billing')->insert([
            'hosting_id' => $hostingId,
            'client_id' => $hosting->userid,
            'billing_cycle' => $hosting->billingcycle,
            'amount' => $product->recurringamount,
            'next_billing_date' => $nextBillingDate,
            'status' => 'active'
        ]);
        
        logActivity("Recurring billing initialized for service #{$hostingId}");
    }
    
    private function calculateNextBillingDate($startDate, $cycle) {
        $now = new \DateTime();
        $next = new \DateTime($startDate);
        
        while ($next <= $now) {
            switch ($cycle) {
                case 'Monthly':
                    $next->modify('+1 month');
                    break;
                case 'Quarterly':
                    $next->modify('+3 months');
                    break;
                case 'SemiAnnually':
                    $next->modify('+6 months');
                    break;
                case 'Annually':
                    $next->modify('+1 year');
                    break;
                case 'Biennially':
                    $next->modify('+2 years');
                    break;
                default:
                    $next->modify('+1 month');
            }
        }
        
        return $next->format('Y-m-d');
    }
    
    public function processRecurringBilling() {
        $today = date('Y-m-d');
        $advanceDays = (int) Capsule::table('tblconfiguration')
            ->where('setting', 'RecurringBillingDaysAdvance')
            ->value('value') ?? 3;
        
        $billableDate = date('Y-m-d', strtotime('+' . $advanceDays . ' days'));
        
        // Get services due for billing
        $dueServices = Capsule::table('mod_recurring_billing')
            ->where('status', 'active')
            ->where('next_billing_date', '<=', $billableDate)
            ->get();
        
        foreach ($dueServices as $service) {
            $this->processRecurringCharge($service);
        }
        
        return [
            'processed' => count($dueServices),
            'successful' => 0,
            'failed' => 0
        ];
    }
    
    private function processRecurringCharge($service) {
        $client = Capsule::table('tblclients')
            ->where('id', $service->client_id)
            ->first();
        
        $hosting = Capsule::table('tblhosting')
            ->where('id', $service->hosting_id)
            ->first();
        
        // Create invoice
        $invoiceId = $this->createInvoice($service);
        
        // Attempt automatic payment
        $paymentResult = $this->attemptPayment($invoiceId, $client);
        
        // Log the attempt
        Capsule::table('mod_recurring_payment_log')->insert([
            'recurring_id' => $service->id,
            'invoice_id' => $invoiceId,
            'status' => $paymentResult['success'] ? 'success' : 'failed',
            'gateway_response' => json_encode($paymentResult)
        ]);
        
        if ($paymentResult['success']) {
            $this->updateNextBillingDate($service);
            return ['success' => true, 'invoice_id' => $invoiceId];
        } else {
            $this->handleFailedPayment($service, $invoiceId, $paymentResult);
            return ['success' => false, 'error' => $paymentResult['error']];
        }
    }
    
    private function createInvoice($service) {
        $client = Capsule::table('tblclients')
            ->where('id', $service->client_id)
            ->first();
        
        $hosting = Capsule::table('tblhosting')
            ->where('id', $service->hosting_id)
            ->first();
        
        $product = Capsule::table('tblproducts')
            ->where('id', $hosting->packageid)
            ->first();
        
        // Create invoice
        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $service->client_id,
            'invoicenum' => '',
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d'),
            'datepaid' => null,
            'status' => 'Unpaid',
            'subtotal' => $service->amount,
            'tax' => 0,
            'tax2' => 0,
            'total' => $service->amount,
            'balance' => $service->amount,
            'paymentmethod' => $client->defaultpaymentmethod
        ]);
        
        // Add line item
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $service->client_id,
            'type' => 'Hosting',
            'relid' => $service->hosting_id,
            'description' => $product->name . ' - ' . ucfirst($service->billing_cycle),
            'amount' => $service->amount
        ]);
        
        return $invoiceId;
    }
    
    private function attemptPayment($invoiceId, $client) {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();
        
        $gateway = $invoice->paymentmethod;
        
        if (empty($gateway)) {
            return ['success' => false, 'error' => 'No payment method'];
        }
        
        // Get saved payment method
        $savedCard = Capsule::table('mod_saved_cards')
            ->where('client_id', $client->id)
            ->where('is_default', true)
            ->first();
        
        if (!$savedCard) {
            return ['success' => false, 'error' => 'No saved payment method'];
        }
        
        // Process payment through gateway
        $result = $this->chargeSavedCard($savedCard, $invoice->total, $gateway);
        
        if ($result['success']) {
            addInvoicePayment($invoiceId, $result['transaction_id'], $invoice->total, 0, $gateway);
            return $result;
        }
        
        return $result;
    }
    
    private function chargeSavedCard($savedCard, $amount, $gateway) {
        // Gateway-specific implementation
        // This would integrate with Stripe, PayPal, etc.
        
        if ($gateway === 'stripe') {
            $stripe = new \Stripe\StripeClient($this->getStripeApiKey());
            
            try {
                $charge = $stripe->charges->create([
                    'amount' => (int)($amount * 100),
                    'currency' => 'usd',
                    'customer' => $savedCard->gateway_customer_id,
                    'payment_method' => $savedCard->payment_method_id
                ]);
                
                return [
                    'success' => $charge->status === 'succeeded',
                    'transaction_id' => $charge->id
                ];
            } catch (Exception $e) {
                return ['success' => false, 'error' => $e->getMessage()];
            }
        }
        
        return ['success' => false, 'error' => 'Unsupported gateway'];
    }
    
    private function updateNextBillingDate($service) {
        $nextDate = $this->calculateNextBillingDate(
            $service->next_billing_date,
            $service->billing_cycle
        );
        
        Capsule::table('mod_recurring_billing')
            ->where('id', $service->id)
            ->update([
                'next_billing_date' => $nextDate,
                'last_billing_date' => date('Y-m-d'),
                'retry_count' => 0
            ]);
    }
    
    private function handleFailedPayment($service, $invoiceId, $result) {
        $retryCount = $service->retry_count + 1;
        $maxRetries = (int) Capsule::table('tblconfiguration')
            ->where('setting', 'MaxPaymentRetries')
            ->value('value') ?? 3;
        
        Capsule::table('mod_recurring_billing')
            ->where('id', $service->id)
            ->update(['retry_count' => $retryCount]);
        
        // Mark invoice overdue
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update(['status' => 'Overdue']);
        
        // Suspend if max retries reached
        if ($retryCount >= $maxRetries) {
            $this->suspendRecurringService($service);
        }
        
        // Send failure notification
        sendTemplatedEmail('RecurringPaymentFailed', $service->client_id, [
            'amount' => $service->amount,
            'retry_date' => date('Y-m-d', strtotime('+3 days')),
            'failure_reason' => $result['error'] ?? 'Payment failed'
        ]);
    }
    
    private function suspendRecurringService($service) {
        Capsule::table('mod_recurring_billing')
            ->where('id', $service->id)
            ->update(['status' => 'suspended']);
        
        Capsule::table('tblhosting')
            ->where('id', $service->hosting_id)
            ->update(['domainstatus' => 'Suspended']);
        
        sendTemplatedEmail('RecurringServiceSuspended', $service->client_id, [
            'amount' => $service->amount,
            'retry_date' => date('Y-m-d', strtotime('+7 days'))
        ]);
        
        logActivity("Recurring billing suspended for service #{$service->hosting_id} after max retries");
    }
}
```

### Step 3: Create Recurring Billing Hooks

Automate recurring billing processes:

```php
// File: /includes/hooks/recurring_billing_automation.php

add_hook('DailyCronJob', 1, function($vars) {
    $billingManager = new \WHMCS\Billing\RecurringBillingManager();
    
    $result = $billingManager->processRecurringBilling();
    
    logActivity("Recurring billing processed: {$result['processed']} services, {$result['successful']} successful, {$result['failed']} failed");
    
    if ($result['failed'] > 0) {
        sendAdminNotification(
            'email',
            'Recurring Billing Report',
            "Processed: {$result['processed']}\nSuccessful: {$result['successful']}\nFailed: {$result['failed']}"
        );
    }
});

add_hook('ServiceCreate', 1, function($vars) {
    $billingManager = new \WHMCS\Billing\RecurringBillingManager();
    $billingManager->initializeRecurringBilling($vars['service_id']);
});

add_hook('InvoicePaid', 1, function($vars) {
    // Check if this was a recurring billing invoice
    $log = Capsule::table('mod_recurring_payment_log')
        ->where('invoice_id', $vars['invoiceid'])
        ->first();
    
    if ($log && $log->status === 'success') {
        // Log the successful recurring payment
        logActivity("Recurring payment successful for invoice #{$vars['invoiceid']}");
    }
});

add_hook('ServiceCancellationRequested', 1, function($vars) {
    // Cancel recurring billing
    Capsule::table('mod_recurring_billing')
        ->where('hosting_id', $vars['service_id'])
        ->update([
            'status' => 'cancelled',
            'cancelled_at' => Capsule::raw('NOW()')
        ]);
});
```

### Step 4: Handle Payment Retry Logic

Implement retry automation:

```php
// File: /includes/hooks/recurring_retry.php

add_hook('DailyCronJob', 1, function($vars) {
    $retryDays = explode(',', Capsule::table('tblconfiguration')
        ->where('setting', 'FailedPaymentRetryDays')
        ->value('value') ?? '3,7,14');
    
    foreach ($retryDays as $dayOffset => $days) {
        $retryDate = date('Y-m-d', strtotime('-' . $days . ' days'));
        
        // Find failed payments due for retry
        $pendingRetries = Capsule::table('mod_recurring_billing')
            ->where('status', 'active')
            ->whereRaw('next_billing_date <= ? AND retry_count = ?', [
                date('Y-m-d', strtotime('-' . $days . ' days')),
                $dayOffset + 1
            ])
            ->get();
        
        foreach ($pendingRetries as $service) {
            // Attempt retry
            $invoice = Capsule::table('tblinvoices')
                ->where('userid', $service->client_id)
                ->where('status', 'Overdue')
                ->orderBy('id', 'desc')
                ->first();
            
            if ($invoice) {
                $client = Capsule::table('tblclients')
                    ->where('id', $service->client_id)
                    ->first();
                
                $billingManager = new \WHMCS\Billing\RecurringBillingManager();
                $result = $billingManager->chargeSavedCard(
                    Capsule::table('mod_saved_cards')
                        ->where('client_id', $client->id)
                        ->where('is_default', true)
                        ->first(),
                    $invoice->total,
                    $invoice->paymentmethod
                );
                
                if ($result['success']) {
                    addInvoicePayment($invoice->id, $result['transaction_id'], $invoice->total, 0, $invoice->paymentmethod);
                    
                    Capsule::table('mod_recurring_billing')
                        ->where('id', $service->id)
                        ->update(['retry_count' => 0, 'status' => 'active']);
                    
                    sendTemplatedEmail('RecurringPaymentRestored', $client->id, [
                        'amount' => $invoice->total
                    ]);
                }
            }
        }
    }
});
```

### Step 5: Generate Recurring Billing Reports

Create revenue and metrics reports:

```php
// File: /includes/hooks/recurring_reporting.php

function generateRecurringBillingReport($startDate, $endDate) {
    $report = [
        'period' => ['start' => $startDate, 'end' => $endDate],
        'summary' => [
            'total_recurring_revenue' => 0,
            'active_subscriptions' => 0,
            'new_subscriptions' => 0,
            'cancelled_subscriptions' => 0,
            'failed_payments' => 0,
            'mrr_churn' => 0
        ],
        'by_billing_cycle' => [],
        'by_product' => []
    ];
    
    // Get active recurring billings
    $recurring = Capsule::table('mod_recurring_billing')
        ->where('status', 'active')
        ->get();
    
    foreach ($recurring as $r) {
        $report['summary']['total_recurring_revenue'] += $r->amount;
        $report['summary']['active_subscriptions']++;
        
        // Group by cycle
        $cycle = $r->billing_cycle;
        if (!isset($report['by_billing_cycle'][$cycle])) {
            $report['by_billing_cycle'][$cycle] = ['count' => 0, 'revenue' => 0];
        }
        $report['by_billing_cycle'][$cycle]['count']++;
        $report['by_billing_cycle'][$cycle]['revenue'] += $r->amount;
    }
    
    // Get new subscriptions
    $report['summary']['new_subscriptions'] = Capsule::table('mod_recurring_billing')
        ->whereBetween('created_at', [$startDate, $endDate])
        ->count();
    
    // Get cancelled subscriptions
    $report['summary']['cancelled_subscriptions'] = Capsule::table('mod_recurring_billing')
        ->where('status', 'cancelled')
        ->whereBetween('cancelled_at', [$startDate, $endDate])
        ->count();
    
    // Get failed payments
    $report['summary']['failed_payments'] = Capsule::table('mod_recurring_payment_log')
        ->where('status', 'failed')
        ->whereBetween('attempted_at', [$startDate, $endDate])
        ->count();
    
    return $report;
}

add_hook('MonthlyCronJob', 1, function($vars) {
    $startDate = date('Y-m-01', strtotime('-1 month'));
    $endDate = date('Y-m-t', strtotime('-1 month'));
    
    $report = generateRecurringBillingReport($startDate, $endDate);
    
    // Save and email report
    $reportText = "Recurring Billing Monthly Report\n";
    $reportText .= "Period: {$startDate} to {$endDate}\n\n";
    $reportText .= "Active Subscriptions: {$report['summary']['active_subscriptions']}\n";
    $reportText .= "Total Recurring Revenue: $" . number_format($report['summary']['total_recurring_revenue'], 2) . "\n";
    $reportText .= "New: {$report['summary']['new_subscriptions']}\n";
    $reportText .= "Cancelled: {$report['summary']['cancelled_subscriptions']}\n";
    $reportText .= "Failed Payments: {$report['summary']['failed_payments']}\n";
    
    sendAdminNotification('email', 'Recurring Billing Report', $reportText);
});
```

## Verification Checklist

- [ ] Recurring billing initialized for products
- [ ] Daily billing processing working
- [ ] Automatic payment collection functioning
- [ ] Failed payment retry logic working
- [ ] Service suspension on max retries
- [ ] Email notifications sent at each stage
- [ ] Billing dates calculated correctly
- [ ] Reports generating accurate data
- [ ] Monthly reports emailed to admin
- [ ] Revenue tracking accurate

## Related Skills and Documentation

- [WHMCS Subscription Workflow](whmcs-subscription-workflow.md)
- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- [WHMCS Overdue Handling](whmcs-overdue-handling-workflow.md)
- WHMCS Documentation: Recurring Billing
- WHMCS Documentation: Cron Automation

## Notes

- Monitor failed payment rates closely
- Implement dunning strategy for retries
- Keep saved payment methods secure
- Test retry logic with various failure scenarios
- Review MRR trends monthly
- Consider early warning for at-risk subscriptions
- Document all payment failures for audit