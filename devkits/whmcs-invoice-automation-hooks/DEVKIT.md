# WHMCS Invoice Automation Hooks Module

## Overview
Comprehensive invoice automation module handling InvoiceCreated, InvoicePaid, and InvoiceCancelled events with workflow automation capabilities.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Invoice Automation Hooks Module
 * 
 * Handles invoice lifecycle events for automation workflows
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 * @license    Commercial License
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: InvoiceCreated
 * Triggered when a new invoice is generated
 */
function whmcs_invoice_automation_invoice_created(array $params): array
{
    try {
        $invoiceId = $params['invoiceid'];
        $userId = $params['userid'];
        $total = $params['total'];
        $now = date('Y-m-d H:i:s');
        
        $config = require __DIR__ . '/config.php';

        logModuleCall(
            'InvoiceAutomation',
            'InvoiceCreated',
            ['invoice_id' => $invoiceId, 'user_id' => $userId, 'total' => $total],
            'Invoice created',
            ''
        );

        // Store invoice creation event
        Capsule::table('mod_invoice_events')->insert([
            'invoice_id' => $invoiceId,
            'event_type' => 'created',
            'event_data' => json_encode([
                'user_id' => $userId,
                'total' => $total,
                'currency' => $params['currency'] ?? 1,
                'payment_method' => $params['paymentmethod'] ?? 'bank_transfer',
            ]),
            'created_at' => $now,
        ]);

        // Auto-apply credits if enabled
        if (!empty($config['auto_apply_credits'])) {
            applyClientCredits($userId, $invoiceId);
        }

        // Send reminder if invoice is overdue (for testing/proforma invoices)
        if (!empty($config['auto_send_initial']) && in_array($params['status'], ['draft', 'proforma'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $invoiceId,
                'type' => 'invoice',
            ];
            localAPI($command, $postData);
        }

        // Trigger custom workflow
        if (!empty($config['workflow_webhook_url'])) {
            wp_remote_post($config['workflow_webhook_url'], [
                'body' => json_encode([
                    'event' => 'invoice.created',
                    'invoice_id' => $invoiceId,
                    'user_id' => $userId,
                    'total' => $total,
                    'due_date' => $params['duedate'] ?? null,
                    'timestamp' => $now,
                ]),
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Webhook-Secret' => $config['webhook_secret'] ?? '',
                ],
            ]);
        }

        // High-value invoice alerts
        if (!empty($config['high_value_threshold']) && $total >= $config['high_value_threshold']) {
            sendAdminNotification('high_value_invoice', [
                'invoice_id' => $invoiceId,
                'user_id' => $userId,
                'total' => $total,
                'threshold' => $config['high_value_threshold'],
            ]);
        }

    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'InvoiceCreated Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: InvoicePaid
 * Triggered when invoice payment is confirmed
 */
function whmcs_invoice_automation_invoice_paid(array $params): array
{
    try {
        $invoiceId = $params['invoiceid'];
        $userId = $params['userid'];
        $amount = $params['amount'];
        $now = date('Y-m-d H:i:s');
        
        $config = require __DIR__ . '/config.php';

        logModuleCall(
            'InvoiceAutomation',
            'InvoicePaid',
            ['invoice_id' => $invoiceId, 'user_id' => $userId, 'amount' => $amount],
            'Invoice paid successfully',
            ''
        );

        // Store payment event
        Capsule::table('mod_invoice_events')->insert([
            'invoice_id' => $invoiceId,
            'event_type' => 'paid',
            'event_data' => json_encode([
                'user_id' => $userId,
                'amount_paid' => $amount,
                'payment_method' => $params['payment_method'] ?? 'unknown',
                'transaction_id' => $params['transaction_id'] ?? null,
                'paid_at' => $now,
            ]),
            'created_at' => $now,
        ]);

        // Update analytics
        Capsule::table('mod_invoice_analytics')->insert([
            'invoice_id' => $invoiceId,
            'user_id' => $userId,
            'invoice_total' => $params['total'] ?? 0,
            'amount_paid' => $amount,
            'payment_method' => $params['payment_method'] ?? 'unknown',
            'days_to_pay' => calculateDaysToPay($invoiceId),
            'paid_at' => $now,
            'created_at' => $now,
        ]);

        // Process referral credits
        if (!empty($config['referral_credit_percent'])) {
            processReferralCredit($userId, $amount, $config['referral_credit_percent']);
        }

        // Send thank you email
        if (!empty($config['send_thank_you_email'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $invoiceId,
                'templatename' => 'Invoice Payment Confirmation',
            ];
            localAPI($command, $postData);
        }

        // Update external systems
        if (!empty($config['accounting_webhook_url'])) {
            wp_remote_post($config['accounting_webhook_url'], [
                'body' => json_encode([
                    'event' => 'invoice.paid',
                    'invoice_id' => $invoiceId,
                    'user_id' => $userId,
                    'amount' => $amount,
                    'payment_date' => $now,
                    'reference' => $params['transaction_id'] ?? null,
                ]),
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Webhook-Secret' => $config['webhook_secret'] ?? '',
                ],
            ]);
        }

        // Unlock suspended services
        if (!empty($config['unlock_on_payment'])) {
            unlockSuspendedServices($userId);
        }

    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'InvoicePaid Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: InvoiceCancelled
 * Triggered when an invoice is cancelled
 */
function whmcs_invoice_automation_invoice_cancelled(array $params): array
{
    try {
        $invoiceId = $params['invoiceid'];
        $userId = $params['userid'];
        $now = date('Y-m-d H:i:s');
        
        $config = require __DIR__ . '/config.php';

        logModuleCall(
            'InvoiceAutomation',
            'InvoiceCancelled',
            ['invoice_id' => $invoiceId, 'user_id' => $userId],
            'Invoice cancelled',
            ''
        );

        // Store cancellation event
        Capsule::table('mod_invoice_events')->insert([
            'invoice_id' => $invoiceId,
            'event_type' => 'cancelled',
            'event_data' => json_encode([
                'user_id' => $userId,
                'cancelled_by' => $params['admin_user'] ?? 'client',
                'reason' => $params['cancel_reason'] ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Release any reserved credits
        if (!empty($config['release_cancelled_credits'])) {
            releaseInvoiceCredits($invoiceId);
        }

        // Notify relevant parties
        if (!empty($config['notify_cancellation'])) {
            sendAdminNotification('invoice_cancelled', [
                'invoice_id' => $invoiceId,
                'user_id' => $userId,
                'cancelled_by' => $params['admin_user'] ?? 'client',
            ]);
        }

    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'InvoiceCancelled Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Apply client credits to invoice
 */
function applyClientCredits(int $userId, int $invoiceId): bool
{
    try {
        $credits = Capsule::table('tblcredit')
            ->where('clientid', $userId)
            ->where('relid', 0)
            ->sum('amount');

        if ($credits > 0) {
            $command = 'AddInvoicePayment';
            $postData = [
                'invoiceid' => $invoiceId,
                'transid' => 'CREDIT-' . time(),
                'gateway' => 'Credit',
                'amount' => min($credits, getInvoiceTotal($invoiceId)),
                'date' => date('Y-m-d H:i:s'),
            ];
            localAPI($command, $postData);
        }

        return true;
    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'ApplyCredits Error', ['user_id' => $userId], $e->getMessage(), '');
        return false;
    }
}

/**
 * Process referral credits for affiliate
 */
function processReferralCredit(int $userId, float $amount, float $percent): void
{
    try {
        $affiliateId = Capsule::table('tblaffiliates')
            ->where('clientid', $userId)
            ->value('id');

        if ($affiliateId) {
            $creditAmount = ($amount * $percent) / 100;
            
            Capsule::table('tblaffiliates')->where('id', $affiliateId)->increment('credit', $creditAmount);
            
            Capsule::table('mod_invoice_referral_credits')->insert([
                'affiliate_id' => $affiliateId,
                'user_id' => $userId,
                'invoice_amount' => $amount,
                'credit_percent' => $percent,
                'credit_amount' => $creditAmount,
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'ReferralCredit Error', ['user_id' => $userId], $e->getMessage(), '');
    }
}

/**
 * Unlock suspended services after payment
 */
function unlockSuspendedServices(int $userId): void
{
    try {
        $suspendedServices = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Suspended')
            ->get(['id', 'packageid']);

        foreach ($suspendedServices as $service) {
            $command = 'ModuleChangePackage';
            $postData = [
                'serviceid' => $service->id,
                'newproductid' => $service->packageid,
            ];
            localAPI($command, $postData);
        }
    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'UnlockServices Error', ['user_id' => $userId], $e->getMessage(), '');
    }
}

/**
 * Release credits from cancelled invoice
 */
function releaseInvoiceCredits(int $invoiceId): void
{
    try {
        $credits = Capsule::table('tblcredit')
            ->where('relid', $invoiceId)
            ->get();

        foreach ($credits as $credit) {
            Capsule::table('tblcredit')
                ->where('id', $credit->id)
                ->update(['relid' => 0]);
        }
    } catch (\Exception $e) {
        logModuleCall('InvoiceAutomation', 'ReleaseCredits Error', ['invoice_id' => $invoiceId], $e->getMessage(), '');
    }
}

/**
 * Calculate days to pay
 */
function calculateDaysToPay(int $invoiceId): int
{
    try {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first(['date', 'duedate']);

        if ($invoice && $invoice->date && $invoice->duedate) {
            $created = new DateTime($invoice->date);
            $due = new DateTime($invoice->duedate);
            return $created->diff($due)->days;
        }
    } catch (\Exception $e) {
        return 0;
    }
    return 0;
}

/**
 * Get invoice total
 */
function getInvoiceTotal(int $invoiceId): float
{
    return Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->value('total') ?? 0;
}

/**
 * Send admin notification
 */
function sendAdminNotification(string $type, array $data): void
{
    $command = 'SendAdminEmail';
    $postData = [
        'type' => $type,
        'customvars' => base64_encode(json_encode($data)),
    ];
    localAPI($command, $postData);
}

// Register hooks
add_hook('InvoiceCreated', 1, 'whmcs_invoice_automation_invoice_created');
add_hook('InvoicePaid', 1, 'whmcs_invoice_automation_invoice_paid');
add_hook('InvoiceCancelled', 1, 'whmcs_invoice_automation_invoice_cancelled');
```

## Configuration File: config.php

```php
<?php
return [
    // Credit Processing
    'auto_apply_credits' => true,
    'release_cancelled_credits' => true,
    'referral_credit_percent' => 10.0,

    // Email Notifications
    'auto_send_initial' => true,
    'send_thank_you_email' => true,
    'notify_cancellation' => true,

    // Service Management
    'unlock_on_payment' => true,

    // Notifications
    'high_value_threshold' => 1000.00,
    'alert_email' => 'billing@hitechcloud.com',

    // Webhook Integration
    'workflow_webhook_url' => '',
    'accounting_webhook_url' => '',
    'webhook_secret' => '',

    // Logging
    'log_level' => 'info',
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Invoice events table
if (!Capsule::schema()->hasTable('mod_invoice_events')) {
    Capsule::schema()->create('mod_invoice_events', function ($table) {
        $table->increments('id');
        $table->integer('invoice_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('invoice_id');
        $table->index('event_type');
        $table->index('created_at');
    });
}

// Invoice analytics table
if (!Capsule::schema()->hasTable('mod_invoice_analytics')) {
    Capsule::schema()->create('mod_invoice_analytics', function ($table) {
        $table->increments('id');
        $table->integer('invoice_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->decimal('invoice_total', 10, 2);
        $table->decimal('amount_paid', 10, 2);
        $table->string('payment_method', 100)->nullable();
        $table->integer('days_to_pay')->default(0);
        $table->timestamp('paid_at')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index(['user_id', 'created_at']);
        $table->index('payment_method');
    });
}

// Referral credits table
if (!Capsule::schema()->hasTable('mod_invoice_referral_credits')) {
    Capsule::schema()->create('mod_invoice_referral_credits', function ($table) {
        $table->increments('id');
        $table->integer('affiliate_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->decimal('invoice_amount', 10, 2);
        $table->decimal('credit_percent', 5, 2);
        $table->decimal('credit_amount', 10, 2);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('affiliate_id');
        $table->index('created_at');
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_invoice_automation_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Invoice Automation Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_invoice_automation_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Invoice Automation Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Priority | Description |
|------|----------|-------------|
| InvoiceCreated | 1 | New invoice generated |
| InvoicePaid | 1 | Payment confirmed |
| InvoiceCancelled | 1 | Invoice cancelled |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
