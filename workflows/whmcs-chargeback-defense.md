# WHMCS Chargeback Defense Workflow

## Description
Respond to payment chargebacks and protect your business.

## Prerequisites
- Payment gateway with dispute handling
- Documentation of transactions
- WHMCS 7.0+

## Steps

### Step 1: Understand Chargeback Reasons
| Code | Reason | Prevention |
|------|--------|------------|
| FR1 | Fraud | 3D Secure, CVV verification |
| FR2 | Product not received | Clear delivery confirmation |
| FR3 | Product unacceptable | Clear descriptions |
| RN1 | Credit not processed | Timely refunds |
| UNC | Unauthorized | Clear authorization |

### Step 2: Create Chargeback Handler
```php
<?php
// Handle chargeback notification

add_hook('PaymentGatewayWebhook', 1, function($vars) {
    if ($vars['type'] === 'chargeback') {
        handleChargeback($vars['data']);
    }
});

function handleChargeback($data)
{
    $transactionId = $data['transaction_id'];
    $amount = $data['amount'];
    $reason = $data['reason'];
    
    // Find associated invoice
    $payment = Capsule::table('tblaccounts')
        ->where('transid', $transactionId)
        ->first();
    
    if ($payment) {
        // Log chargeback
        Capsule::table('mod_chargebacks')->insert([
            'invoice_id' => $payment->invoiceid,
            'user_id' => $payment->userid,
            'transaction_id' => $transactionId,
            'amount' => $amount,
            'reason' => $reason,
            'status' => 'disputed',
            'due_date' => date('Y-m-d', strtotime('+10 days')),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Suspend service temporarily
        $services = Capsule::table('tblhosting')
            ->where('userid', $payment->userid)
            ->where('domainstatus', 'Active')
            ->get();
        
        foreach ($services as $service) {
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Suspended']);
        }
        
        // Send notification
        sendEmail('ChargebackReceived', $payment->userid, [
            'amount' => $amount,
            'reason' => $reason,
        ]);
        
        logActivity("Chargeback received: Transaction $transactionId, Amount $amount");
    }
}
```

### Step 3: Prepare Dispute Evidence
```php
<?php
function prepareChargebackEvidence($chargebackId)
{
    $chargeback = Capsule::table('mod_chargebacks')->find($chargebackId);
    $invoice = Capsule::table('tblinvoices')->find($chargeback->invoice_id);
    
    $evidence = [
        'chargeback_id' => $chargebackId,
        'transaction_id' => $chargeback->transaction_id,
        
        // Customer info
        'customer_email' => getClientEmail($chargeback->user_id),
        'customer_name' => getClientName($chargeback->user_id),
        'customer_ip' => getCustomerIP($chargeback->user_id),
        
        // Transaction details
        'invoice_amount' => $invoice->total,
        'invoice_date' => $invoice->date,
        'invoice_items' => getInvoiceItems($invoice->id),
        
        // Product/service delivery
        'service_created' => getServiceCreationDate($invoice->id),
        'delivery_confirmation' => getDeliveryConfirmation($invoice->id),
        
        // Communication history
        'support_tickets' => getSupportTickets($chargeback->user_id),
        'email_history' => getEmailHistory($chargeback->user_id),
    ];
    
    return $evidence;
}
```

### Step 4: Submit Dispute
```php
<?php
function submitChargebackDispute($chargebackId, $evidence)
{
    $chargeback = Capsule::table('mod_chargebacks')->find($chargebackId);
    
    // Call gateway dispute API
    $result = $gateway->submitDispute([
        'chargeback_id' => $chargeback->chargeback_id,
        'evidence' => $evidence,
    ]);
    
    if ($result['success']) {
        Capsule::table('mod_chargebacks')
            ->where('id', $chargebackId)
            ->update([
                'status' => 'evidence_submitted',
                'dispute_id' => $result['dispute_id'],
            ]);
    }
    
    return $result;
}
```

### Step 5: Post-Resolution Handling
```php
<?php
function handleChargebackResolution($chargebackId, $outcome)
{
    $chargeback = Capsule::table('mod_chargebacks')->find($chargebackId);
    
    if ($outcome === 'won') {
        // Restore service
        Capsule::table('tblhosting')
            ->where('userid', $chargeback->user_id)
            ->update(['domainstatus' => 'Active']);
        
        logActivity("Chargeback resolved - Win: $chargeback->transaction_id");
    } else {
        // Customer wins - service stays suspended or terminated
        logActivity("Chargeback resolved - Loss: $chargeback->transaction_id");
    }
    
    Capsule::table('mod_chargebacks')
        ->where('id', $chargebackId)
        ->update([
            'status' => $outcome,
            'resolved_at' => date('Y-m-d H:i:s'),
        ]);
}
```

## Prevention Strategies
1. Clear product descriptions
2. Delivery confirmation
3. Responsive support
4. Easy refund process
5. 3D Secure authentication
6. Address verification

## Tags
- chargeback
- dispute
- fraud
- payment