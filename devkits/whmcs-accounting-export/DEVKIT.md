# WHMCS Accounting Export Module

## Overview
Accounting software export module for QuickBooks, Xero, FreshBooks, and other platforms.

## Module File: accounting_export.php

```php
<?php
/**
 * WHMCS Accounting Export Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_Accounting_Export
{
    protected $config;
    protected $supportedFormats = ['quickbooks', 'xero', 'freshbooks', 'csv', 'json'];

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Export invoices to accounting format
     */
    public function exportInvoices(array $params): array
    {
        $format = $params['format'] ?? 'csv';
        $startDate = $params['start_date'] ?? date('Y-01-01');
        $endDate = $params['end_date'] ?? date('Y-m-d');

        if (!in_array($format, $this->supportedFormats)) {
            return ['success' => false, 'error' => 'Unsupported format'];
        }

        $invoices = Capsule::table('tblinvoices')
            ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
            ->whereBetween('tblinvoices.date', [$startDate, $endDate])
            ->whereIn('tblinvoices.status', ['Paid', 'Partial'])
            ->get([
                'tblinvoices.*',
                'tblclients.email',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
            ]);

        $data = $this->formatInvoicesForExport($invoices);

        $exportData = $this->formatByType($data, $format);

        // Store export record
        $exportId = Capsule::table('mod_accounting_exports')->insertGetId([
            'export_type' => 'invoices',
            'format' => $format,
            'record_count' => count($data),
            'date_from' => $startDate,
            'date_to' => $endDate,
            'file_data' => $this->config['store_files'] ? json_encode($exportData) : null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return [
            'success' => true,
            'export_id' => $exportId,
            'record_count' => count($data),
            'data' => $exportData,
        ];
    }

    /**
     * Export transactions
     */
    public function exportTransactions(array $params): array
    {
        $format = $params['format'] ?? 'csv';
        $startDate = $params['start_date'] ?? date('Y-01-01');
        $endDate = $params['end_date'] ?? date('Y-m-d');

        $transactions = Capsule::table('tblaccounts')
            ->join('tblclients', 'tblaccounts.userid', '=', 'tblclients.id')
            ->join('tblinvoices', 'tblaccounts.invoiceid', '=', 'tblinvoices.id', 'left')
            ->whereBetween('tblaccounts.date', [$startDate, $endDate])
            ->get([
                'tblaccounts.*',
                'tblclients.email',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
                'tblinvoices.invoicenum',
            ]);

        $data = $this->formatTransactionsForExport($transactions);

        return [
            'success' => true,
            'record_count' => count($data),
            'data' => $this->formatByType($data, $format),
        ];
    }

    /**
     * Format invoices for export
     */
    protected function formatInvoicesForExport(array $invoices): array
    {
        $data = [];

        foreach ($invoices as $invoice) {
            $data[] = [
                'invoice_id' => $invoice->id,
                'invoice_number' => $invoice->invoicenum ?? $invoice->id,
                'date' => $invoice->date,
                'due_date' => $invoice->duedate,
                'client_id' => $invoice->userid,
                'client_name' => $invoice->client_name,
                'client_email' => $invoice->email,
                'subtotal' => $invoice->subtotal,
                'tax' => $invoice->tax,
                'tax2' => $invoice->tax2,
                'total' => $invoice->total,
                'status' => $invoice->status,
                'payment_method' => $invoice->paymentmethod,
            ];
        }

        return $data;
    }

    /**
     * Format transactions for export
     */
    protected function formatTransactionsForExport(array $transactions): array
    {
        $data = [];

        foreach ($transactions as $tx) {
            $data[] = [
                'transaction_id' => $tx->id,
                'date' => $tx->date,
                'client_id' => $tx->userid,
                'client_name' => $tx->client_name,
                'invoice_number' => $tx->invoicenum,
                'description' => $tx->description,
                'amount_in' => $tx->amountin,
                'amount_out' => $tx->amountout,
                'fees' => $tx->fees,
                'gateway' => $tx->gateway,
            ];
        }

        return $data;
    }

    /**
     * Format data by export type
     */
    protected function formatByType(array $data, string $format): string
    {
        switch ($format) {
            case 'csv':
                return $this->toCSV($data);
            case 'json':
                return json_encode($data, JSON_PRETTY_PRINT);
            case 'quickbooks':
                return $this->toQuickBooksFormat($data);
            case 'xero':
                return $this->toXeroFormat($data);
            case 'freshbooks':
                return $this->toFreshBooksFormat($data);
            default:
                return json_encode($data);
        }
    }

    protected function toCSV(array $data): string
    {
        if (empty($data)) {
            return '';
        }

        $headers = array_keys($data[0]);
        $csv = implode(',', $headers) . "\n";

        foreach ($data as $row) {
            $values = array_map(function($val) {
                return '"' . str_replace('"', '""', $val) . '"';
            }, array_values($row));
            $csv .= implode(',', $values) . "\n";
        }

        return $csv;
    }

    protected function toQuickBooksFormat(array $data): string
    {
        $output = ['Invoice' => []];
        foreach ($data as $invoice) {
            $output['Invoice'][] = [
                'TxnDate' => $invoice['date'],
                'RefNumber' => $invoice['invoice_number'],
                'CustomerName' => $invoice['client_name'],
                'DueDate' => $invoice['due_date'],
                'Amount' => $invoice['total'],
            ];
        }
        return json_encode($output);
    }

    protected function toXeroFormat(array $data): string
    {
        $output = ['Invoices' => []];
        foreach ($data as $invoice) {
            $output['Invoices'][] = [
                'InvoiceNumber' => $invoice['invoice_number'],
                'Date' => $invoice['date'],
                'DueDate' => $invoice['due_date'],
                'ContactName' => $invoice['client_name'],
                'Total' => $invoice['total'],
            ];
        }
        return json_encode($output);
    }

    protected function toFreshBooksFormat(array $data): string
    {
        $output = ['invoices' => []];
        foreach ($data as $invoice) {
            $output['invoices'][] = [
                'number' => $invoice['invoice_number'],
                'date' => $invoice['date'],
                'client_email' => $invoice['client_email'],
                'amount' => $invoice['total'],
            ];
        }
        return json_encode($output);
    }

    /**
     * Get export history
     */
    public function getExportHistory(): array
    {
        return Capsule::table('mod_accounting_exports')
            ->orderBy('created_at', 'desc')
            ->limit(20)
            ->get();
    }
}

function whmcs_accounting_export_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_accounting_exports')) {
            Capsule::schema()->create('mod_accounting_exports', function ($table) {
                $table->increments('id');
                $table->string('export_type', 50);
                $table->string('format', 20);
                $table->integer('record_count');
                $table->date('date_from');
                $table->date('date_to');
                $table->longText('file_data')->nullable();
                $table->timestamp('created_at')->useCurrent();
            });
        }
        return ['status' => 'success', 'description' => 'Accounting Export activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_accounting_export_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Accounting Export deactivated'];
}

function whmcs_accounting_export_config(): array
{
    return [
        'default_format' => ['FriendlyName' => 'Default Format', 'Type' => 'dropdown', 'Options' => 'csv,json,quickbooks,xero,freshbooks'],
        'store_files' => ['FriendlyName' => 'Store Export Files', 'Type' => 'yesno', 'Description' => 'Store exported files in database'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'default_format' => 'csv',
    'store_files' => false,
    'export_path' => '/exports/',
];
```
