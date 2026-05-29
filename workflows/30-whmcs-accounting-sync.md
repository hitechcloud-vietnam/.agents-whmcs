# WHMCS Accounting Sync Workflow

## Overview
This workflow covers synchronizing WHMCS financial data with accounting systems.

## Step 1: Accounting Sync Service

```php
<?php
// src/Service/AccountingSyncService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class AccountingSyncService
{
    private $accountingApi;

    public function __construct(ExternalApiService $accountingApi)
    {
        $this->accountingApi = $accountingApi;
    }

    public function syncInvoice(int $invoiceId): array
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if (!$invoice) {
            return ['success' => false, 'error' => 'Invoice not found'];
        }

        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();
        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->get();

        $accountingData = [
            'invoice_number' => $invoice->invoicenum ?? 'INV-' . $invoice->id,
            'type' => 'invoice',
            'status' => $invoice->status,
            'date' => $invoice->date,
            'due_date' => $invoice->duedate,
            'client' => [
                'name' => $client->firstname . ' ' . $client->lastname,
                'email' => $client->email,
                'vat_id' => $client->tax_id ?? null
            ],
            'line_items' => [],
            'subtotal' => $invoice->subtotal,
            'tax' => $invoice->tax,
            'total' => $invoice->total
        ];

        foreach ($items as $item) {
            $accountingData['line_items'][] = [
                'description' => $item->description,
                'type' => $item->type,
                'amount' => $item->amount,
                'taxable' => $item->taxed
            ];
        }

        try {
            $result = $this->accountingApi->post('/invoices', $accountingData);

            // Update sync status
            Capsule::table('mod_accounting_sync')
                ->updateOrInsert(
                    ['invoice_id' => $invoiceId],
                    [
                        'accounting_id' => $result['id'],
                        'synced_at' => date('Y-m-d H:i:s'),
                        'sync_type' => 'invoice'
                    ]
                );

            return ['success' => true, 'accounting_id' => $result['id']];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    public function syncPayment(int $invoiceId, string $transactionId): array
    {
        $payment = Capsule::table('tblaccounts')
            ->where('invoiceid', $invoiceId)
            ->where('transactionid', $transactionId)
            ->first();

        if (!$payment) {
            return ['success' => false, 'error' => 'Payment not found'];
        }

        // Find accounting invoice ID
        $syncRecord = Capsule::table('mod_accounting_sync')
            ->where('invoice_id', $invoiceId)
            ->first();

        if (!$syncRecord) {
            return ['success' => false, 'error' => 'Invoice not synced'];
        }

        $paymentData = [
            'invoice_id' => $syncRecord->accounting_id,
            'amount' => $payment->amountin,
            'payment_method' => $payment->paymentmethod,
            'transaction_id' => $payment->transactionid,
            'date' => $payment->date
        ];

        try {
            $result = $this->accountingApi->post('/payments', $paymentData);

            Capsule::table('mod_accounting_sync')
                ->updateOrInsert(
                    ['transaction_id' => $transactionId],
                    [
                        'accounting_id' => $result['id'],
                        'synced_at' => date('Y-m-d H:i:s'),
                        'sync_type' => 'payment'
                    ]
                );

            return ['success' => true, 'accounting_id' => $result['id']];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    public function syncRefund(int $invoiceId, float $amount): array
    {
        $refundData = [
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'date' => date('Y-m-d')
        ];

        try {
            $result = $this->accountingApi->post('/refunds', $refundData);
            return ['success' => true, 'accounting_id' => $result['id']];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    public function generateFinancialReport(string $startDate, string $endDate): array
    {
        $invoices = Capsule::table('tblinvoices')
            ->whereBetween('date', [$startDate, $endDate])
            ->get();

        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'invoices' => [
                'total_count' => 0,
                'total_amount' => 0,
                'paid_count' => 0,
                'paid_amount' => 0,
                'unpaid_count' => 0,
                'unpaid_amount' => 0
            ],
            'payments' => [
                'total_count' => 0,
                'total_amount' => 0
            ]
        ];

        foreach ($invoices as $invoice) {
            $report['invoices']['total_count']++;
            $report['invoices']['total_amount'] += $invoice->total;

            if ($invoice->status === 'Paid') {
                $report['invoices']['paid_count']++;
                $report['invoices']['paid_amount'] += $invoice->total;
            } elseif ($invoice->status === 'Unpaid') {
                $report['invoices']['unpaid_count']++;
                $report['invoices']['unpaid_amount'] += $invoice->total;
            }
        }

        $payments = Capsule::table('tblaccounts')
            ->whereBetween('date', [$startDate, $endDate])
            ->where('amountin', '>', 0)
            ->get();

        foreach ($payments as $payment) {
            $report['payments']['total_count']++;
            $report['payments']['total_amount'] += $payment->amountin;
        }

        return $report;
    }
}
```

## Step 2: Accounting Sync Cron

```php
<?php
// includes/cron/accounting_sync_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\AccountingSyncService;

$accountingApi = new \WHMCS\Module\Addon\YourModule\Service\ExternalApiService(
    Capsule::config('accounting_api_url'),
    Capsule::config('accounting_api_key')
);

$syncService = new AccountingSyncService($accountingApi);

// Sync unpaid invoices
$unpaidInvoices = Capsule::table('tblinvoices')
    ->where('status', 'Unpaid')
    ->whereNull('deleted')
    ->get();

foreach ($unpaidInvoices as $invoice) {
    $result = $syncService->syncInvoice($invoice->id);
    echo "Invoice {$invoice->id}: " . ($result['success'] ? 'Synced' : 'Failed') . "\n";
}

// Sync recent payments
$recentPayments = Capsule::table('tblaccounts')
    ->where('date', '>=', date('Y-m-d', strtotime('-24 hours')))
    ->where('amountin', '>', 0)
    ->get();

foreach ($recentPayments as $payment) {
    $result = $syncService->syncPayment($payment->invoiceid, $payment->transactionid);
    echo "Payment {$payment->transactionid}: " . ($result['success'] ? 'Synced' : 'Failed') . "\n";
}
```

## Verification Checklist

- [ ] Accounting service implemented
- [ ] Invoice sync working
- [ ] Payment sync working
- [ ] Refund sync working
- [ ] Financial reports generating
- [ ] Cron job configured
- [ ] Test sync completed successfully
