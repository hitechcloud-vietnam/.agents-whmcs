# WHMCS Refund Processing Workflow

## Overview
This workflow automates and standardizes the refund process in WHMCS.

## Step 1: Refund Service

```php
<?php
// src/Service/RefundService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class RefundService
{
    public function processRefund(int $invoiceId, float $amount, string $reason, string $refundMethod): array
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if (!$invoice) {
            return ['success' => false, 'error' => 'Invoice not found'];
        }

        if ($amount > $invoice->total) {
            return ['success' => false, 'error' => 'Refund amount exceeds invoice total'];
        }

        // Create refund record
        $refundId = Capsule::table('mod_refunds')->insertGetId([
            'invoice_id' => $invoiceId,
            'client_id' => $invoice->userid,
            'amount' => $amount,
            'reason' => $reason,
            'method' => $refundMethod,
            'status' => 'pending',
            'requested_at' => date('Y-m-d H:i:s'),
            'requested_by' => $_SESSION['adminid'] ?? null
        ]);

        // Process refund based on method
        $result = match ($refundMethod) {
            'original_payment' => $this->refundToOriginalPayment($invoice, $amount),
            'credit' => $this->refundToCredit($invoice->userid, $amount),
            'bank_transfer' => $this->processBankRefund($invoice, $amount),
            default => ['success' => false, 'error' => 'Unknown refund method']
        };

        if ($result['success']) {
            Capsule::table('mod_refunds')
                ->where('id', $refundId)
                ->update([
                    'status' => 'completed',
                    'transaction_id' => $result['transaction_id'] ?? null,
                    'processed_at' => date('Y-m-d H:i:s')
                ]);

            // Create credit note
            $this->createCreditNote($invoiceId, $amount);
        } else {
            Capsule::table('mod_refunds')
                ->where('id', $refundId)
                ->update([
                    'status' => 'failed',
                    'error_message' => $result['error'] ?? 'Unknown error'
                ]);
        }

        return $result;
    }

    private function refundToOriginalPayment($invoice, float $amount): array
    {
        // Get original payment details
        $payment = Capsule::table('tblaccounts')
            ->where('invoiceid', $invoice->id)
            ->where('amountin', '>', 0)
            ->first();

        if (!$payment) {
            return ['success' => false, 'error' => 'Original payment not found'];
        }

        // Process refund through payment gateway
        $gateway = Capsule::module('gateway')->load($payment->paymentmethod);

        try {
            $result = $gateway->refund([
                'transaction_id' => $payment->transactionid,
                'amount' => $amount
            ]);

            return [
                'success' => true,
                'transaction_id' => $result['refund_id'] ?? null
            ];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function refundToCredit(int $clientId, float $amount): array
    {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->increment('credit', $amount);

        Capsule::table('mod_refund_credit_log')->insert([
            'client_id' => $clientId,
            'amount' => $amount,
            'type' => 'refund',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return ['success' => true];
    }

    private function processBankRefund($invoice, float $amount): array
    {
        // Mark as pending bank transfer
        return [
            'success' => true,
            'status' => 'pending_bank_transfer'
        ];
    }

    private function createCreditNote(int $invoiceId, float $amount): void
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        Capsule::table('tblcreditnotes')->insert([
            'userid' => $invoice->userid,
            'invoice_id' => $invoiceId,
            'credit' => $amount,
            'status' => 'Active',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Verification Checklist

- [ ] Refund service implemented
- [ ] Refund methods configured
- [ ] Credit note creation working
- [ ] Transaction logging working
