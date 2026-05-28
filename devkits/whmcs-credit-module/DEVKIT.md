# WHMCS Credit Module DevKit

## Purpose
Store credit management system allowing clients to deposit funds, accumulate credits, and apply them to future invoices automatically or manually.

## Module Type
Billing/Finance Addon Module

## Use Cases
- Prepaid account balances
- Account credit for overpayments
- Manual credit additions by admin
- Credit-based billing (pay with credit first)
- Customer loyalty rewards

## Database Schema

```sql
-- Client credit balances
CREATE TABLE `mod_credits` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `client_id` INT UNSIGNED NOT NULL,
  `currency_id` INT UNSIGNED NOT NULL DEFAULT 1,
  `balance` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `total_deposited` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `total_used` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `last_updated` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_client_currency` (`client_id`, `currency_id`),
  FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE,
  FOREIGN KEY (`currency_id`) REFERENCES `tblcurrencies`(`id`)
) ENGINE=InnoDB;

-- Credit transactions log
CREATE TABLE `mod_credit_transactions` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `credit_id` INT UNSIGNED NOT NULL,
  `type` ENUM('deposit', 'withdrawal', 'payment', 'refund', 'adjustment', 'expiry') NOT NULL,
  `amount` DECIMAL(10,2) NOT NULL,
  `balance_after` DECIMAL(10,2) NOT NULL,
  `reference_type` ENUM('invoice', 'refund', 'manual', 'promotion', 'expiry') NULL,
  `reference_id` INT UNSIGNED NULL,
  `description` VARCHAR(500) NULL,
  `admin_id` INT UNSIGNED NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`credit_id`) REFERENCES `mod_credits`(`id`) ON DELETE CASCADE,
  FOREIGN KEY (`admin_id`) REFERENCES `tbladmins`(`id`) ON DELETE SET NULL,
  INDEX `idx_credit_type` (`credit_id`, `type`),
  INDEX `idx_created` (`created_at`)
) ENGINE=InnoDB;

-- Credit limits per client
CREATE TABLE `mod_credit_limits` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `client_id` INT UNSIGNED NOT NULL UNIQUE,
  `credit_limit` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `notify_threshold_percent` TINYINT UNSIGNED NOT NULL DEFAULT 75,
  `is_active` TINYINT(1) NOT NULL DEFAULT 1,
  FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB;
```

## Code Template

### Main Module File (credit.php)

```php
<?php
/**
 * WHMCS Credit Module
 *
 * @package WHMCS
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Module meta data
 */
function credit_MetaData()
{
    return [
        'DisplayName' => 'Credit Module',
        'RequiresServer' => false,
    ];
}

/**
 * Module configuration
 */
function credit_config()
{
    return [
        'name' => 'Credit System',
        'description' => 'Store credit management for client balances',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'autoDeduct' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Automatically deduct credit when generating invoices',
            ],
            'allowNegative' => [
                'Type' => 'yesno',
                'Default' => 'off',
                'Description' => 'Allow negative balance (credit limit)',
            ],
            'expiryDays' => [
                'Type' => 'text',
                'Default' => '365',
                'Description' => 'Credit expiry days (0 = never)',
            ],
            'minDeposit' => [
                'Type' => 'text',
                'Default' => '10.00',
                'Description' => 'Minimum deposit amount',
            ],
        ],
    ];
}

/**
 * Activate module
 */
function credit_activate()
{
    $tables = [
        "CREATE TABLE IF NOT EXISTS `mod_credits` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `client_id` INT UNSIGNED NOT NULL,
            `currency_id` INT UNSIGNED NOT NULL DEFAULT 1,
            `balance` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
            `total_deposited` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
            `total_used` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
            `last_updated` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            UNIQUE KEY `uk_client_currency` (`client_id`, `currency_id`),
            FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
        ) ENGINE=InnoDB",

        "CREATE TABLE IF NOT EXISTS `mod_credit_transactions` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `credit_id` INT UNSIGNED NOT NULL,
            `type` ENUM('deposit', 'withdrawal', 'payment', 'refund', 'adjustment', 'expiry') NOT NULL,
            `amount` DECIMAL(10,2) NOT NULL,
            `balance_after` DECIMAL(10,2) NOT NULL,
            `reference_type` ENUM('invoice', 'refund', 'manual', 'promotion', 'expiry') NULL,
            `reference_id` INT UNSIGNED NULL,
            `description` VARCHAR(500) NULL,
            `admin_id` INT UNSIGNED NULL,
            `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (`credit_id`) REFERENCES `mod_credits`(`id`) ON DELETE CASCADE,
            INDEX `idx_credit_type` (`credit_id`, `type`),
            INDEX `idx_created` (`created_at`)
        ) ENGINE=InnoDB",

        "CREATE TABLE IF NOT EXISTS `mod_credit_limits` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `client_id` INT UNSIGNED NOT NULL UNIQUE,
            `credit_limit` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
            `notify_threshold_percent` TINYINT UNSIGNED NOT NULL DEFAULT 75,
            `is_active` TINYINT(1) NOT NULL DEFAULT 1,
            FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
        ) ENGINE=InnoDB"
    ];

    try {
        foreach ($tables as $query) {
            full_query($query);
        }
        return ['status' => 'success', 'description' => 'Credit module activated'];
    } catch (Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

/**
 * Deactivate module
 */
function credit_deactivate()
{
    try {
        full_query("DROP TABLE IF EXISTS `mod_credit_transactions`");
        full_query("DROP TABLE IF EXISTS `mod_credit_limits`");
        full_query("DROP TABLE IF EXISTS `mod_credits`");
        return ['status' => 'success'];
    } catch (Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}
```

### Credit Service Class

```php
<?php
/**
 * Credit Service Class
 */

namespace WHMCS\Module\Credit;

use WHMCS\Database\Capsule;

class CreditService
{
    /**
     * Get or create client credit account
     */
    public static function getClientCredit($clientId, $currencyId = 1)
    {
        $credit = Capsule::table('mod_credits')
            ->where('client_id', $clientId)
            ->where('currency_id', $currencyId)
            ->first();

        if (!$credit) {
            // Create new credit account
            $id = Capsule::table('mod_credits')->insertGetId([
                'client_id' => $clientId,
                'currency_id' => $currencyId,
                'balance' => 0.00,
                'total_deposited' => 0.00,
                'total_used' => 0.00,
            ]);
            $credit = Capsule::table('mod_credits')->find($id);
        }

        return $credit;
    }

    /**
     * Add credit to client account
     */
    public static function addCredit($clientId, $amount, $type = 'deposit', $description = '', $referenceId = null, $adminId = null)
    {
        $currencyId = self::getClientCurrency($clientId);
        $credit = self::getClientCredit($clientId, $currencyId);

        $newBalance = bcadd($credit->balance, $amount, 2);

        // Update balance
        Capsule::table('mod_credits')
            ->where('id', $credit->id)
            ->update([
                'balance' => $newBalance,
                'total_deposited' => bcadd($credit->total_deposited, $amount, 2),
            ]);

        // Log transaction
        Capsule::table('mod_credit_transactions')->insert([
            'credit_id' => $credit->id,
            'type' => $type,
            'amount' => $amount,
            'balance_after' => $newBalance,
            'reference_type' => ($type === 'deposit' ? 'manual' : null),
            'reference_id' => $referenceId,
            'description' => $description,
            'admin_id' => $adminId,
        ]);

        // Notify if threshold reached
        self::checkCreditLimitNotification($clientId);

        return ['success' => true, 'balance' => $newBalance];
    }

    /**
     * Deduct credit from client account
     */
    public static function deductCredit($clientId, $amount, $type = 'payment', $description = '', $referenceId = null)
    {
        $currencyId = self::getClientCurrency($clientId);
        $credit = self::getClientCredit($clientId, $currencyId);

        $newBalance = bcsub($credit->balance, $amount, 2);

        // Check if negative allowed
        $allowNegative = Capsule::table('mod_credit_config')->first();
        if (!$allowNegative && $newBalance < 0) {
            return ['success' => false, 'message' => 'Insufficient credit balance'];
        }

        // Update balance
        Capsule::table('mod_credits')
            ->where('id', $credit->id)
            ->update([
                'balance' => $newBalance,
                'total_used' => bcadd($credit->total_used, $amount, 2),
            ]);

        // Log transaction
        Capsule::table('mod_credit_transactions')->insert([
            'credit_id' => $credit->id,
            'type' => $type,
            'amount' => -$amount,
            'balance_after' => $newBalance,
            'reference_type' => ($type === 'payment' ? 'invoice' : null),
            'reference_id' => $referenceId,
            'description' => $description,
        ]);

        return ['success' => true, 'balance' => $newBalance];
    }

    /**
     * Apply credit to invoice
     */
    public static function applyToInvoice($invoiceId, $clientId = null)
    {
        if (!$clientId) {
            $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
            $clientId = $invoice->userid;
        }

        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        if ($invoice->status !== 'Unpaid') {
            return ['success' => false, 'message' => 'Invoice is not unpaid'];
        }

        $credit = self::getClientCredit($clientId);
        $balance = $invoice->total - $invoice->amount_paid;
        $amountToApply = min($credit->balance, $balance);

        if ($amountToApply <= 0) {
            return ['success' => false, 'message' => 'No credit available'];
        }

        // Deduct from credit
        self::deductCredit($clientId, $amountToApply, 'payment', "Applied to invoice #{$invoiceId}", $invoiceId);

        // Update invoice
        $newAmountPaid = bcadd($invoice->amount_paid, $amountToApply, 2);
        $newStatus = ($newAmountPaid >= $invoice->total) ? 'Paid' : 'PaymentPending';

        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update([
                'amount_paid' => $newAmountPaid,
                'status' => $newStatus,
            ]);

        // Log credit usage
        logTransaction('Credit Module', [
            'invoice_id' => $invoiceId,
            'amount' => $amountToApply,
        ], 'Applied');

        return [
            'success' => true,
            'amount_applied' => $amountToApply,
            'new_balance' => $credit->balance - $amountToApply,
            'invoice_status' => $newStatus,
        ];
    }

    /**
     * Get transaction history
     */
    public static function getTransactionHistory($clientId, $limit = 50, $offset = 0)
    {
        $credit = self::getClientCredit($clientId);

        return Capsule::table('mod_credit_transactions')
            ->where('credit_id', $credit->id)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get();
    }

    /**
     * Get client currency
     */
    protected static function getClientCurrency($clientId)
    {
        $client = Capsule::table('tblclients')->find($clientId);
        return $client->currency ?? 1;
    }

    /**
     * Check credit limit notification
     */
    protected static function checkCreditLimitNotification($clientId)
    {
        $credit = self::getClientCredit($clientId);
        $limit = Capsule::table('mod_credit_limits')
            ->where('client_id', $clientId)
            ->where('is_active', 1)
            ->first();

        if (!$limit || $limit->credit_limit <= 0) {
            return;
        }

        $thresholdAmount = $limit->credit_limit * ($limit->notify_threshold_percent / 100);

        if ($credit->balance >= $thresholdAmount) {
            // Send notification
            $client = Capsule::table('tblclients')->find($clientId);
            sendMessage('Credit Threshold Notification', $clientId, [
                'credit_balance' => formatCurrency($credit->balance),
                'credit_limit' => formatCurrency($limit->credit_limit),
            ]);
        }
    }

    /**
     * Transfer credit between clients
     */
    public static function transferCredit($fromClientId, $toClientId, $amount, $description = '')
    {
        // Deduct from sender
        $deductResult = self::deductCredit($fromClientId, $amount, 'withdrawal', "Transfer to client #{$toClientId}");
        if (!$deductResult['success']) {
            return $deductResult;
        }

        // Add to recipient
        self::addCredit($toClientId, $amount, 'deposit', "Transfer from client #{$fromClientId}. {$description}");

        return ['success' => true, 'message' => "Transferred {$amount} successfully"];
    }
}
```

## Hook Integrations

### PreInvoiceCreation - Auto-apply credit

```php
<?php
/**
 * Hook: PreInvoiceCreation - Auto-apply credit balance
 */
add_hook('PreInvoiceCreation', 1, function($vars) {
    $clientId = $vars['userid'];
    if (!$clientId) {
        return;
    }

    $credit = \WHMCS\Module\Credit\CreditService::getClientCredit($clientId);
    if ($credit && $credit->balance > 0) {
        $_SESSION['apply_credit_to_invoice'] = true;
        $_SESSION['credit_to_apply'] = $credit->balance;
    }
});
```

### InvoicePaid - Mark credit payment

```php
<?php
/**
 * Hook: InvoicePaid - Log credit payment
 */
add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')->find($invoiceId);

    if ($invoice->paymentmethod === 'credit') {
        logActivity("Invoice #{$invoiceId} paid using store credit");
    }
});
```

### AdminClientSummaryTabs - Credit tab

```php
<?php
/**
 * Hook: AdminClientSummaryTabs - Add credit info tab
 */
add_hook('AdminClientSummaryTabs', 1, function($vars) {
    $clientId = $vars['userid'];
    $credit = \WHMCS\Module\Credit\CreditService::getClientCredit($clientId);
    $transactions = \WHMCS\Module\Credit\CreditService::getTransactionHistory($clientId, 10);

    return [
        'credits' => [
            'tabname' => 'Store Credit',
            'label' => 'Credit',
            'uri' => 'clientsummary.php?userid=' . $clientId . '&tab=credit',
            'displayName' => formatCurrency($credit->balance),
        ],
    ];
});
```

## Admin Templates

### Admin Credit Management Page

```php
<?php
/**
 * Admin Credit Management - admin/credit.php
 */

use WHMCS\Module\Credit\CreditService;

define('ADMINAREA', true);
require("../../../init.php");

if (!check_permission('Client Management', true)) {
    redir("accessdenied");
}

$action = $_REQUEST['action'] ?? 'view';
$clientId = (int)($_REQUEST['userid'] ?? 0);

switch ($action) {
    case 'add':
        check_token('WHMCS.admin.default');
        $amount = (float)$_POST['amount'];
        $description = $_POST['description'] ?? '';

        if ($amount <= 0) {
            flash('error', 'Amount must be positive');
            redir("admin/credit.php?action=view&userid={$clientId}");
        }

        $result = CreditService::addCredit(
            $clientId,
            $amount,
            'deposit',
            $description,
            null,
            $_SESSION['adminid']
        );

        if ($result['success']) {
            logActivity("Added {$amount} credit to client #{$clientId}", $_SESSION['adminid']);
            flash('success', 'Credit added successfully');
        } else {
            flash('error', $result['message'] ?? 'Failed to add credit');
        }
        redir("admin/credit.php?action=view&userid={$clientId}");
        break;

    case 'deduct':
        check_token('WHMCS.admin.default');
        $amount = (float)$_POST['amount'];
        $description = $_POST['description'] ?? '';

        $result = CreditService::deductCredit(
            $clientId,
            $amount,
            'adjustment',
            $description . ' (Admin adjustment)',
            null
        );

        if ($result['success']) {
            logActivity("Deducted {$amount} credit from client #{$clientId}", $_SESSION['adminid']);
            flash('success', 'Credit deducted successfully');
        } else {
            flash('error', $result['message'] ?? 'Failed to deduct credit');
        }
        redir("admin/credit.php?action=view&userid={$clientId}");
        break;

    case 'apply':
        check_token('WHMCS.admin.default');
        $invoiceId = (int)$_POST['invoice_id'];
        $result = CreditService::applyToInvoice($invoiceId, $clientId);

        if ($result['success']) {
            flash('success', "Applied {$result['amount_applied']} credit to invoice");
        } else {
            flash('error', $result['message'] ?? 'Failed to apply credit');
        }
        redir("admin/credit.php?action=view&userid={$clientId}");
        break;
}

// Fetch client credit info
$credit = CreditService::getClientCredit($clientId);
$transactions = CreditService::getTransactionHistory($clientId, 50);

// Fetch unpaid invoices
$unpaidInvoices = Capsule::table('tblinvoices')
    ->where('userid', $clientId)
    ->whereIn('status', ['Unpaid', 'PaymentPending'])
    ->orderBy('duedate', 'asc')
    ->get();

$smartyvalues = [
    'clientId' => $clientId,
    'credit' => $credit,
    'transactions' => $transactions,
    'unpaidInvoices' => $unpaidInvoices,
];
```

### Client Area Credit Page

```php
<?php
/**
 * Client Area Credit Page - client/credit.php
 */

use WHMCS\Module\Credit\CreditService;

define('CLIENTAREA', true);
require("../../../init.php");

$pageTitle = "Store Credit";

if (!$_SESSION['uid']) {
    header("Location: login.php");
    exit;
}

$clientId = $_SESSION['uid'];

// Handle deposit request
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_POST['do'] === 'deposit') {
    check_token('WHMCS.default');

    $amount = (float)$_POST['amount'];
    $minDeposit = Capsule::table('mod_credit_config')->first()['min_deposit'] ?? 10;

    if ($amount < $minDeposit) {
        $error = "Minimum deposit amount is " . formatCurrency($minDeposit);
    } else {
        // Create payment
        $invoiceId = CreditService::createDepositInvoice($clientId, $amount);
        if ($invoiceId) {
            header("Location: viewinvoice.php?id={$invoiceId}");
            exit;
        }
    }
}

// Fetch credit balance
$credit = CreditService::getClientCredit($clientId);
$transactions = CreditService::getTransactionHistory($clientId, 20);

$smartyvalues = [
    'pagetitle' => $pageTitle,
    'creditBalance' => $credit->balance,
    'totalDeposited' => $credit->total_deposited,
    'totalUsed' => $credit->total_used,
    'transactions' => $transactions,
];
```

## Development Checklist

### Phase 1: Core Credit System
- [ ] Create database tables
- [ ] Implement CreditService class
- [ ] Build get/add/deduct credit functions
- [ ] Add transaction logging
- [ ] Test credit balance calculations

### Phase 2: Invoice Integration
- [ ] Auto-apply credit hook
- [ ] Manual credit application
- [ ] Partial payment handling
- [ ] Invoice status updates
- [ ] Credit on invoice PDF

### Phase 3: Admin Interface
- [ ] Admin credit view page
- [ ] Add/deduct credit forms
- [ ] Transaction history display
- [ ] Bulk credit operations
- [ ] Client credit limits

### Phase 4: Client Interface
- [ ] Client credit dashboard
- [ ] Transaction history
- [ ] Deposit request form
- [ ] Credit transfer feature
- [ ] Notification settings

### Phase 5: Advanced Features
- [ ] Credit expiry system
- [ ] Credit limits per client
- [ ] Threshold notifications
- [ ] Credit promotions
- [ ] Refund to credit option

### Phase 6: Security & Compliance
- [ ] Admin permission checks
- [ ] Input validation
- [ ] Audit logging
- [ ] Anti-fraud measures
- [ ] Data export/privacy

### Phase 7: Testing
- [ ] Unit tests for calculations
- [ ] Concurrent access handling
- [ ] Negative balance scenarios
- [ ] Currency conversion
- [ ] Integration tests
