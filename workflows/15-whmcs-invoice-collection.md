# WHMCS Invoice Collection Workflow

## Overview
This workflow automates invoice collection efforts, including dunning sequences, payment reminders, and collection actions.

## Step 1: Invoice Collection Service

```php
<?php
// src/Service/InvoiceCollectionService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class InvoiceCollectionService
{
    private $dunningSequence = [
        1 => ['days' => 1, 'template' => 'Invoice Payment Reminder'],
        2 => ['days' => 3, 'template' => 'Invoice Second Reminder'],
        3 => ['days' => 7, 'template' => 'Invoice Third Reminder'],
        4 => ['days' => 10, 'template' => 'Invoice Fourth Reminder'],
        5 => ['days' => 14, 'template' => 'Invoice Final Notice'],
    ];

    public function processCollectionQueue(): array
    {
        $results = [
            'reminders_sent' => 0,
            'escalations' => 0,
            'late_fees_added' => 0
        ];

        $overdueInvoices = $this->getOverdueInvoices();

        foreach ($overdueInvoices as $invoice) {
            $daysOverdue = $this->calculateDaysOverdue($invoice->duedate);

            // Determine dunning stage
            $stage = $this->determineDunningStage($daysOverdue);

            // Check if reminder should be sent
            if ($this->shouldSendReminder($invoice, $stage)) {
                $this->sendCollectionReminder($invoice, $stage);
                $results['reminders_sent']++;
            }

            // Add late fee if applicable
            if ($stage >= 3 && !$this->lateFeeApplied($invoice->id)) {
                $this->applyLateFee($invoice);
                $results['late_fees_added']++;
            }

            // Escalate if needed
            if ($stage >= 5) {
                $this->escalateToCollection($invoice);
                $results['escalations']++;
            }
        }

        return $results;
    }

    private function getOverdueInvoices(): array
    {
        return Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->where('duedate', '<', date('Y-m-d'))
            ->where('deleted', '!=', 1)
            ->where('collection_stage', '<', 5)
            ->get();
    }

    private function calculateDaysOverdue(string $dueDate): int
    {
        $due = new \DateTime($dueDate);
        $today = new \DateTime();
        return max(0, $today->diff($due)->days);
    }

    private function determineDunningStage(int $daysOverdue): int
    {
        foreach ($this->dunningSequence as $stage => $config) {
            if ($daysOverdue >= $config['days']) {
                return $stage;
            }
        }
        return 1;
    }

    private function shouldSendReminder($invoice, int $stage): bool
    {
        // Check if reminder already sent for this stage
        $existingReminder = Capsule::table('mod_collection_reminders')
            ->where('invoice_id', $invoice->id)
            ->where('stage', $stage)
            ->first();

        return !$existingReminder;
    }

    private function sendCollectionReminder($invoice, int $stage): void
    {
        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();
        $template = $this->dunningSequence[$stage]['template'];

        send_email($template, $client->email, [
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'invoice_id' => $invoice->id,
            'amount_due' => formatCurrency($invoice->total),
            'due_date' => fromMySQLDate($invoice->duedate),
            'days_overdue' => $this->calculateDaysOverdue($invoice->duedate),
            'payment_url' => Capsule::config('system_url') . '/viewinvoice.php?id=' . $invoice->id
        ]);

        // Log reminder
        Capsule::table('mod_collection_reminders')->insert([
            'invoice_id' => $invoice->id,
            'client_id' => $invoice->userid,
            'stage' => $stage,
            'template_used' => $template,
            'sent_at' => date('Y-m-d H:i:s')
        ]);

        // Update invoice stage
        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->update(['collection_stage' => $stage]);
    }

    private function lateFeeApplied(int $invoiceId): bool
    {
        $feeItem = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->where('type', 'LateFee')
            ->first();

        return $feeItem !== null;
    }

    private function applyLateFee($invoice): void
    {
        $lateFeeAmount = $this->calculateLateFee($invoice);

        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoice->id,
            'userid' => $invoice->userid,
            'type' => 'LateFee',
            'description' => 'Late Payment Fee',
            'amount' => $lateFeeAmount,
            'taxed' => 0
        ]);

        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->increment('total', $lateFeeAmount);
    }

    private function calculateLateFee($invoice): float
    {
        $daysOverdue = $this->calculateDaysOverdue($invoice->duedate);

        // 2% late fee after 7 days, max $50
        if ($daysOverdue >= 7) {
            $fee = $invoice->total * 0.02;
            return min($fee, 50);
        }

        return 0;
    }

    private function escalateToCollection($invoice): void
    {
        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->update(['collection_stage' => 5, 'in_collection' => 1]);

        // Log collection escalation
        Capsule::table('mod_collection_escalations')->insert([
            'invoice_id' => $invoice->id,
            'client_id' => $invoice->userid,
            'amount' => $invoice->total,
            'escalated_at' => date('Y-m-d H:i:s'),
            'status' => 'pending'
        ]);

        // Notify admin
        $adminEmail = Capsule::config('admin_email');
        send_email('Collection Escalation Alert', $adminEmail, [
            'invoice_id' => $invoice->id,
            'client_id' => $invoice->userid,
            'amount' => formatCurrency($invoice->total),
            'days_overdue' => $this->calculateDaysOverdue($invoice->duedate)
        ]);
    }
}
```

## Step 2: Collection Cron Job

```php
<?php
// includes/cron/collection_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\InvoiceCollectionService;

echo "=== Invoice Collection Cron ===\n";

$service = new InvoiceCollectionService();
$results = $service->processCollectionQueue();

echo "Reminders sent: {$results['reminders_sent']}\n";
echo "Escalations: {$results['escalations']}\n";
echo "Late fees added: {$results['late_fees_added']}\n";

echo "=== Collection Cron Complete ===\n";
```

## Verification Checklist

- [ ] Collection service implemented
- [ ] Dunning sequence configured
- [ ] Reminder emails working
- [ ] Late fee calculation correct
- [ ] Escalation logic working
- [ ] Cron job configured
- [ ] Test collection run completed
