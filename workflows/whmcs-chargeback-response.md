# WHMCS Chargeback Response Workflow

## Overview
This workflow automates chargeback dispute responses.

## Prerequisites
- Payment gateway integration
- Document storage

## Step-by-Step Guide

### Step 1: Create Chargeback Handler
```php
add_hook('ChargebackCreated', 1, function($vars) {
    $chargebackId = $vars['chargebackId'];
    $transactionId = $vars['transactionId'];
    
    // Get transaction details
    $transaction = get_transaction($transactionId);
    $client = get_client($transaction->userid);
    $service = get_service_by_invoice($transaction->invoiceid);
    
    // Create dispute record
    \WHMCS\Database\Capsule::table('mod_yourmodule_chargebacks')->insert([
        'chargeback_id' => $chargebackId,
        'transaction_id' => $transactionId,
        'client_id' => $transaction->userid,
        'amount' => $vars['amount'],
        'reason' => $vars['reason'],
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Notify admin
    notify_admin_chargeback($chargebackId);
    
    // Gather evidence
    $evidence = collect_chargeback_evidence($transaction, $client, $service);
    
    return [
        'success' => true,
        'evidence' => $evidence,
    ];
});

function collect_chargeback_evidence($transaction, $client, $service): array
{
    return [
        'customer_communication' => get_client_emails($client->id),
        'proof_of_delivery' => get_delivery_confirmation($service->id),
        'contract' => get_signed_contract($transaction->invoiceid),
        'transaction_receipt' => get_invoice_pdf($transaction->invoiceid),
    ];
}
```

## Chargeback Response Checklist

### Detection
- [ ] Chargeback identified
- [ ] Client notified
- [ ] Admin alerted

### Evidence Collection
- [ ] Communications gathered
- [ ] Delivery proof collected
- [ ] Contract retrieved

### Submission
- [ ] Evidence compiled
- [ ] Response submitted
- [ ] Status tracked
