# WHMCS Billing Automation Workflow

## Overview
This workflow automates billing operations in WHMCS, including invoice generation, payment processing, and overdue account handling.

## Billing Flow

```
Service Due → Invoice Generated → Payment Due → Overdue → Suspension → Termination
```

## Step 1: Configure Billing Hooks

```php
<?php
// includes/hooks/billing_hook.php

use WHMCS\Database\Capsule;

// Hook: Before invoice creation
add_hook('InvoiceCreationPreEmail', 1, function($params) {
    $invoiceId = $params['invoice_id'];

    // Add custom line items
    $customItems = Capsule::table('mod_billing_custom_items')
        ->where('client_id', $params['user_id'])
        ->where('applied', 0)
        ->get();

    foreach ($customItems as $item) {
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $params['user_id'],
            'type' => $item->item_type,
            'description' => $item->description,
            'amount' => $item->amount,
            'taxed' => $item->taxed
        ]);

        Capsule::table('mod_billing_custom_items')
            ->where('id', $item->id)
            ->update(['applied' => 1, 'invoice_id' => $invoiceId]);
    }

    return $params;
});

// Hook: After payment received
add_hook('AfterPaymentReceived', 1, function($params) {
    $invoiceId = $params['invoice_id'];
    $amount = $params['amount'];

    // Log payment for analytics
    Capsule::table('mod_billing_payments')->insert([
        'invoice_id' => $invoiceId,
        'amount' => $amount,
        'payment_method' => $params['payment_method'],
        'transaction_id' => $params['transaction_id'],
        'received_at' => date('Y-m-d H:i:s')
    ]);

    // Check for overpayment
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
    if ($amount > $invoice->total) {
        $overpayment = $amount - $invoice->total;
        handleOverpayment($params['user_id'], $overpayment);
    }

    return $params;
});

// Hook: Invoice overdue
add_hook('InvoiceOverdue', 1, function($params) {
    $invoiceId = $params['invoice_id'];
    $daysOverdue = $params['days_overdue'];

    // Log overdue event
    logActivity("Invoice #{$invoiceId} is {$daysOverdue} days overdue");

    // Take action based on days overdue
    if ($daysOverdue >= 7) {
        run_hook('SuspendOverdueService', ['invoice_id' => $invoiceId]);
    }

    if ($daysOverdue >= 14) {
        run_hook('TerminateOverdueService', ['invoice_id' => $invoiceId]);
    }

    return $params;
});
```

## Step 2: Automated Invoice Generation

```php
<?php
// src/Service/InvoiceAutomationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;
use WHMCS\Module\Addon\YourModule\Helper\DateHelper;

class InvoiceAutomationService
{
    private $dateHelper;

    public function __construct()
    {
        $this->dateHelper = new DateHelper();
    }

    public function generateRecurringInvoices(): array
    {
        $generated = 0;
        $failed = 0;

        // Get services due for billing
        $dueServices = $this->getDueServices();

        foreach ($dueServices as $service) {
            try {
                $this->generateInvoiceForService($service);
                $generated++;
            } catch (\Exception $e) {
                $failed++;
                logActivity("Invoice generation failed for service {$service->id}: " . $e->getMessage());
            }
        }

        return [
            'generated' => $generated,
            'failed' => $failed
        ];
    }

    private function getDueServices(): array
    {
        $today = date('Y-m-d');

        return Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.nextduedate', '<=', $today)
            ->whereIn('tblhosting.billingcycle', ['Monthly', 'Quarterly', 'Semi-Annual', 'Annual'])
            ->where('tblhosting.domainstatus', 'Active')
            ->where('tblclients.status', 'Active')
            ->where(function($query) {
                $query->whereNull('tblhosting.regdate')
                    ->orWhere('tblhosting.regdate', '!=', $today);
            })
            ->get();
    }

    private function generateInvoiceForService($service): int
    {
        // Check if invoice already exists for this due date
        $existingInvoice = Capsule::table('tblinvoices')
            ->join('tblinvoiceitems', 'tblinvoices.id', '=', 'tblinvoiceitems.invoiceid')
            ->where('tblinvoices.userid', $service->userid)
            ->where('tblinvoiceitems.relid', $service->id)
            ->where('tblinvoices.status', '!=', 'Cancelled')
            ->where('tblinvoices.duedate', $service->nextduedate)
            ->first();

        if ($existingInvoice) {
            throw new \Exception("Invoice already exists for service {$service->id}");
        }

        // Calculate amount
        $amount = $this->calculateServiceAmount($service);

        // Create invoice
        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $service->userid,
            'invoicenum' => $this->getNextInvoiceNumber(),
            'date' => date('Y-m-d'),
            'duedate' => $service->nextduedate,
            'datepaid' => null,
            'status' => 'Unpaid',
            'subtotal' => $amount,
            'tax' => 0,
            'total' => $amount,
            'paymentmethod' => $service->paymentmethod ?: 'invoice',
            'notes' => "Recurring billing for {$service->domain}"
        ]);

        // Add line item
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $service->userid,
            'type' => 'Hosting',
            'relid' => $service->id,
            'description' => $this->getServiceDescription($service),
            'amount' => $amount,
            'taxed' => 1
        ]);

        // Apply taxes
        $this->applyTaxes($invoiceId, $service->userid);

        // Update next due date
        $newDueDate = $this->calculateNextDueDate($service->nextduedate, $service->billingcycle);

        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update(['nextduedate' => $newDueDate]);

        // Send invoice notification
        $this->sendInvoiceNotification($invoiceId);

        return $invoiceId;
    }

    private function calculateServiceAmount($service): float
    {
        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $service->packageid)
            ->first();

        $cycle = strtolower($service->billingcycle);

        $monthly = $pricing->monthly ?? 0;
        $quarterly = $pricing->quarterly ?? 0;
        $semiannually = $pricing->semiannually ?? 0;
        $annually = $pricing->annually ?? 0;

        switch ($cycle) {
            case 'monthly':
                return (float)$monthly;
            case 'quarterly':
                return (float)$quarterly;
            case 'semiannually':
                return (float)$semiannually;
            case 'annually':
                return (float)$annually;
            default:
                return (float)$monthly;
        }
    }

    private function applyTaxes(int $invoiceId, int $userId): void
    {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        $country = $client->country;
        $state = $client->state;

        // Get applicable taxes
        $taxRates = $this->getApplicableTaxRates($country, $state);

        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $subtotal = $invoice->subtotal;

        $totalTax = 0;
        foreach ($taxRates as $rate) {
            $taxAmount = $subtotal * ($rate['rate'] / 100);
            $totalTax += $taxAmount;

            Capsule::table('tblinvoiceitems')->insert([
                'invoiceid' => $invoiceId,
                'userid' => $userId,
                'type' => 'Tax',
                'description' => $rate['name'] . ' (' . $rate['rate'] . '%)',
                'amount' => $taxAmount,
                'taxed' => 0
            ]);
        }

        // Update invoice totals
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update([
                'tax' => $totalTax,
                'total' => $subtotal + $totalTax
            ]);
    }

    private function getApplicableTaxRates(string $country, string $state): array
    {
        $rates = [];

        $taxRules = Capsule::table('tbltax')
            ->where('country', $country)
            ->where(function($query) use ($state) {
                $query->where('state', $state)
                    ->orWhereNull('state')
                    ->orWhere('state', '');
            })
            ->get();

        foreach ($taxRules as $rule) {
            $rates[] = [
                'name' => $rule->name,
                'rate' => $rule->taxrate
            ];
        }

        return $rates;
    }

    private function calculateNextDueDate(string $currentDueDate, string $billingCycle): string
    {
        $date = new \DateTime($currentDueDate);

        switch (strtolower($billingCycle)) {
            case 'monthly':
                $date->modify('+1 month');
                break;
            case 'quarterly':
                $date->modify('+3 months');
                break;
            case 'semiannually':
                $date->modify('+6 months');
                break;
            case 'annually':
                $date->modify('+1 year');
                break;
        }

        return $date->format('Y-m-d');
    }

    private function getServiceDescription($service): string
    {
        $product = Capsule::table('tblproducts')
            ->where('id', $service->packageid)
            ->first();

        return $product->name . ' - ' . $service->domain;
    }

    private function getNextInvoiceNumber(): string
    {
        $lastInvoice = Capsule::table('tblinvoices')
            ->orderBy('id', 'desc')
            ->first();

        $lastNum = $lastInvoice ? (int)substr($lastInvoice->invoicenum, 4) : 0;
        return 'INV-' . str_pad($lastNum + 1, 6, '0', STR_PAD_LEFT);
    }

    private function sendInvoiceNotification(int $invoiceId): void
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();

        send_email('Invoice Created', $client->email, [
            'invoice_id' => $invoiceId,
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'amount' => formatCurrency($invoice->total),
            'due_date' => fromMySQLDate($invoice->duedate)
        ]);
    }
}
```

## Step 3: Overdue Account Handling

```php
<?php
// src/Service/OverdueAccountService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class OverdueAccountService
{
    private $suspensionDays = 7;
    private $terminationDays = 14;
    private $finalReminderDays = 3;

    public function processOverdueAccounts(): array
    {
        $today = date('Y-m-d');
        $results = [
            'reminders_sent' => 0,
            'services_suspended' => 0,
            'services_terminated' => 0
        ];

        // Get overdue invoices
        $overdueInvoices = $this->getOverdueInvoices();

        foreach ($overdueInvoices as $invoice) {
            $daysOverdue = $this->calculateDaysOverdue($invoice->duedate);

            // Send reminders at specific intervals
            if ($this->shouldSendReminder($invoice, $daysOverdue)) {
                $this->sendOverdueReminder($invoice, $daysOverdue);
                $results['reminders_sent']++;
            }

            // Suspend services after configured days
            if ($daysOverdue >= $this->suspensionDays && $invoice->suspended != 1) {
                $this->suspendServices($invoice);
                $results['services_suspended']++;
            }

            // Terminate after configured days
            if ($daysOverdue >= $this->terminationDays && $invoice->terminated != 1) {
                $this->terminateServices($invoice);
                $results['services_terminated']++;
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
            ->get();
    }

    private function calculateDaysOverdue(string $dueDate): int
    {
        $due = new \DateTime($dueDate);
        $today = new \DateTime();
        return $today->diff($due)->days;
    }

    private function shouldSendReminder($invoice, int $daysOverdue): bool
    {
        // Send reminders at 1, 3, 7, 10, 14 days
        $reminderDays = [1, 3, 7, 10, 14];
        return in_array($daysOverdue, $reminderDays);
    }

    private function sendOverdueReminder($invoice, int $daysOverdue): void
    {
        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();

        $template = $this->getReminderTemplate($daysOverdue);

        send_email($template, $client->email, [
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'invoice_id' => $invoice->id,
            'amount_due' => formatCurrency($invoice->total),
            'due_date' => fromMySQLDate($invoice->duedate),
            'days_overdue' => $daysOverdue,
            'payment_url' => Capsule::config('system_url') . '/viewinvoice.php?id=' . $invoice->id
        ]);

        // Log reminder
        Capsule::table('mod_billing_reminders')->insert([
            'invoice_id' => $invoice->id,
            'client_id' => $invoice->userid,
            'reminder_type' => $this->getReminderTemplate($daysOverdue),
            'days_overdue' => $daysOverdue,
            'sent_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function getReminderTemplate(int $daysOverdue): string
    {
        if ($daysOverdue >= 14) {
            return 'Overdue Invoice Final Notice';
        } elseif ($daysOverdue >= 7) {
            return 'Overdue Invoice Second Reminder';
        } else {
            return 'Overdue Invoice Reminder';
        }
    }

    private function suspendServices($invoice): void
    {
        // Get services linked to this invoice
        $serviceIds = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoice->id)
            ->where('type', 'Hosting')
            ->pluck('relid')
            ->toArray();

        foreach ($serviceIds as $serviceId) {
            $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

            if ($service && $service->domainstatus === 'Active') {
                // Call server module suspend function
                $this->executeServiceSuspension($service);
            }
        }

        // Update invoice flag
        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->update(['suspended' => 1]);
    }

    private function executeServiceSuspension($service): void
    {
        if (!$service->serverid) {
            return;
        }

        $server = Capsule::table('tblservers')->where('id', $service->serverid)->first();

        if (!$server) {
            return;
        }

        // Load server module
        $module = new \WHMCS\Module\Server($server->type);
        $module->setServerCredential('hostname', $server->hostname);
        $module->setServerCredential('username', $server->username);
        $module->setServerCredential('password', decrypt($server->password));

        try {
            $module->call('SuspendAccount', [
                'serviceid' => $service->id
            ]);

            // Update service status
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Suspended']);

            logActivity("Service {$service->id} suspended for overdue invoice");

        } catch (\Exception $e) {
            logActivity("Failed to suspend service {$service->id}: " . $e->getMessage());
        }
    }

    private function terminateServices($invoice): void
    {
        $serviceIds = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoice->id)
            ->where('type', 'Hosting')
            ->pluck('relid')
            ->toArray();

        foreach ($serviceIds as $serviceId) {
            $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

            if ($service && in_array($service->domainstatus, ['Active', 'Suspended'])) {
                $this->executeServiceTermination($service);
            }
        }

        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->update(['terminated' => 1]);
    }

    private function executeServiceTermination($service): void
    {
        if (!$service->serverid) {
            return;
        }

        $server = Capsule::table('tblservers')->where('id', $service->serverid)->first();

        if (!$server) {
            return;
        }

        $module = new \WHMCS\Module\Server($server->type);
        $module->setServerCredential('hostname', $server->hostname);
        $module->setServerCredential('username', $server->username);
        $module->setServerCredential('password', decrypt($server->password));

        try {
            $module->call('TerminateAccount', [
                'serviceid' => $service->id
            ]);

            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Terminated']);

            logActivity("Service {$service->id} terminated for overdue invoice");

        } catch (\Exception $e) {
            logActivity("Failed to terminate service {$service->id}: " . $e->getMessage());
        }
    }
}
```

## Step 4: Billing Cron Jobs

```php
<?php
// includes/cron/billing_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\InvoiceAutomationService;
use WHMCS\Module\Addon\YourModule\Service\OverdueAccountService;

echo "=== Billing Cron Job ===\n";
echo "Started: " . date('Y-m-d H:i:s') . "\n\n";

// Generate recurring invoices
echo "Generating recurring invoices...\n";
$invoiceService = new InvoiceAutomationService();
$invoiceResult = $invoiceService->generateRecurringInvoices();
echo "Generated: {$invoiceResult['generated']}, Failed: {$invoiceResult['failed']}\n";

// Process overdue accounts
echo "Processing overdue accounts...\n";
$overdueService = new OverdueAccountService();
$overdueResult = $overdueService->processOverdueAccounts();
echo "Reminders sent: {$overdueResult['reminders_sent']}\n";
echo "Services suspended: {$overdueResult['services_suspended']}\n";
echo "Services terminated: {$overdueResult['services_terminated']}\n";

echo "\n=== Billing Cron Complete ===\n";
```

## Step 5: Payment Processing

```php
<?php
// src/Service/PaymentProcessingService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class PaymentProcessingService
{
    public function processPayment(int $invoiceId, string $paymentMethod, array $gatewayData = []): array
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if (!$invoice) {
            return ['success' => false, 'error' => 'Invoice not found'];
        }

        if ($invoice->status === 'Paid') {
            return ['success' => false, 'error' => 'Invoice already paid'];
        }

        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();

        try {
            switch ($paymentMethod) {
                case 'creditcard':
                    return $this->processCreditCard($invoice, $client, $gatewayData);
                case 'paypal':
                    return $this->processPayPal($invoice, $client, $gatewayData);
                case 'banktransfer':
                    return $this->processBankTransfer($invoice, $client);
                case 'credit':
                    return $this->applyAccountCredit($invoice, $client);
                default:
                    return ['success' => false, 'error' => 'Unknown payment method'];
            }
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function processCreditCard($invoice, $client, array $gatewayData): array
    {
        // Initialize payment gateway
        $gateway = Capsule::module('gateway')->load('stripe');

        // Process payment through gateway
        $result = $gateway->capture($gatewayData);

        if ($result['success']) {
            $this->recordPayment($invoice->id, $result['amount'], 'creditcard', $result['transaction_id']);
            $this->applyPayment($invoice->id);

            return [
                'success' => true,
                'transaction_id' => $result['transaction_id']
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'] ?? 'Payment failed'
        ];
    }

    private function processPayPal($invoice, $client, array $gatewayData): array
    {
        // Initiate PayPal checkout
        $approvalUrl = $this->initiatePayPalCheckout($invoice, $client);

        return [
            'success' => true,
            'redirect_url' => $approvalUrl
        ];
    }

    private function processBankTransfer($invoice, $client): array
    {
        // Generate bank transfer details
        $transferDetails = $this->generateBankTransferDetails($invoice);

        // Send bank transfer instructions
        send_email('Bank Transfer Instructions', $client->email, [
            'invoice_id' => $invoice->id,
            'amount' => formatCurrency($invoice->total),
            'bank_details' => $transferDetails
        ]);

        return [
            'success' => true,
            'instructions_sent' => true
        ];
    }

    private function applyAccountCredit($invoice, $client): array
    {
        $creditBalance = $this->getClientCreditBalance($client->id);

        if ($creditBalance < $invoice->total) {
            return [
                'success' => false,
                'error' => 'Insufficient credit balance',
                'available_credit' => $creditBalance
            ];
        }

        // Deduct from credit
        $newBalance = $creditBalance - $invoice->total;

        Capsule::table('tblclients')
            ->where('id', $client->id)
            ->update(['credit' => $newBalance]);

        // Record credit usage
        Capsule::table('mod_billing_credit_log')->insert([
            'client_id' => $client->id,
            'invoice_id' => $invoice->id,
            'amount' => $invoice->total,
            'balance_before' => $creditBalance,
            'balance_after' => $newBalance,
            'used_at' => date('Y-m-d H:i:s')
        ]);

        $this->recordPayment($invoice->id, $invoice->total, 'credit', null);
        $this->applyPayment($invoice->id);

        return [
            'success' => true,
            'amount_applied' => $invoice->total,
            'remaining_credit' => $newBalance
        ];
    }

    private function recordPayment(int $invoiceId, float $amount, string $method, ?string $transactionId): void
    {
        Capsule::table('tblaccounts')->insert([
            'userid' => Capsule::table('tblinvoices')->where('id', $invoiceId)->value('userid'),
            'invoiceid' => $invoiceId,
            'date' => date('Y-m-d H:i:s'),
            'description' => 'Payment Received',
            'amountin' => $amount,
            'amountout' => 0,
            'paymentmethod' => $method,
            'transactionid' => $transactionId
        ]);
    }

    private function applyPayment(int $invoiceId): void
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $amountPaid = Capsule::table('tblaccounts')
            ->where('invoiceid', $invoiceId)
            ->sum('amountin');

        $status = $amountPaid >= $invoice->total ? 'Paid' : 'Partial';

        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update([
                'status' => $status,
                'datepaid' => $status === 'Paid' ? date('Y-m-d H:i:s') : null
            ]);

        // Activate services if fully paid
        if ($status === 'Paid') {
            $this->activatePaidServices($invoiceId);
        }
    }

    private function activatePaidServices(int $invoiceId): void
    {
        $serviceIds = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->where('type', 'Hosting')
            ->pluck('relid')
            ->toArray();

        foreach ($serviceIds as $serviceId) {
            $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

            if ($service && $service->domainstatus === 'Suspended') {
                Capsule::table('tblhosting')
                    ->where('id', $serviceId)
                    ->update(['domainstatus' => 'Active']);

                // Unsuspend on server
                run_hook('ServiceUnsuspended', ['service_id' => $serviceId]);
            }
        }
    }

    private function getClientCreditBalance(int $clientId): float
    {
        return (float)Capsule::table('tblclients')
            ->where('id', $clientId)
            ->value('credit');
    }
}
```

## Verification Checklist

- [ ] Billing hooks registered
- [ ] Invoice automation service implemented
- [ ] Recurring invoice generation working
- [ ] Tax calculation implemented
- [ ] Overdue handling service created
- [ ] Suspension workflow tested
- [ ] Termination workflow tested
- [ ] Payment processing service implemented
- [ ] Credit handling working
- [ ] Cron jobs configured
- [ ] Email templates configured
- [ ] Test billing cycle completed
