# WHMCS Credit Management Workflow

## Purpose

Manage client credit balances, handle credit additions, apply credits to invoices, and maintain accurate credit accounting records.

## Prerequisites

- WHMCS with client credit functionality
- Admin access to billing operations
- Credit policies documented
- Integration with accounting system (optional)

## Workflow Steps

### Step 1: Configure Credit Settings

Set up credit management configuration:

```php
// Add credit configuration to database
INSERT INTO tblconfiguration (setting, value) VALUES 
('CreditEnabled', 'on'),
('CreditMinThreshold', '5.00'),
('CreditAutoApply', 'on'),
('CreditAutoApplyThreshold', '10.00'),
('CreditMaxPerTransaction', '500.00');

// Create credit tracking table
Capsule::schema()->create('mod_credit_transactions', function($t) {
    $t->increments('id');
    $t->integer('client_id');
    $t->decimal('amount', 10, 2);
    $t->string('type'); // add, remove, apply, refund
    $t->string('reference'); // invoice_id or transaction_id
    $t->string('description');
    $t->integer('admin_id');
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Credit Management Functions

Implement core credit operations:

```php
// File: /includes/classes/CreditManager.php

namespace WHMCS\Billing;

class CreditManager {
    
    public function addCredit($clientId, $amount, $description, $adminId = null, $reference = null) {
        // Validate amount
        if ($amount <= 0) {
            throw new Exception('Credit amount must be positive');
        }
        
        // Check maximum per transaction
        $maxTransaction = $this->getSetting('CreditMaxPerTransaction', 500);
        if ($amount > $maxTransaction) {
            throw new Exception("Credit amount exceeds maximum of {$maxTransaction}");
        }
        
        // Get current balance
        $currentBalance = $this->getBalance($clientId);
        $newBalance = $currentBalance + $amount;
        
        // Add credit to client account
        Capsule::table('tblcredit')
            ->insert([
                'clientid' => $clientId,
                'amount' => $amount,
                'remaining' => $newBalance,
                'date' => date('Y-m-d H:i:s'),
                'description' => $description
            ]);
        
        // Record transaction
        Capsule::table('mod_credit_transactions')->insert([
            'client_id' => $clientId,
            'amount' => $amount,
            'type' => 'add',
            'reference' => $reference,
            'description' => $description,
            'admin_id' => $adminId ?? $_SESSION['adminid']
        ]);
        
        // Log the credit addition
        logActivity("Credit of {$amount} added for client #{$clientId}: {$description}");
        
        // Send notification if significant
        if ($amount >= 50) {
            sendTemplatedEmail('CreditAdded', $clientId, [
                'amount' => $amount,
                'new_balance' => $newBalance,
                'description' => $description
            ]);
        }
        
        return ['success' => true, 'new_balance' => $newBalance];
    }
    
    public function removeCredit($clientId, $amount, $description, $adminId = null) {
        if ($amount <= 0) {
            throw new Exception('Credit removal amount must be positive');
        }
        
        $currentBalance = $this->getBalance($clientId);
        if ($amount > $currentBalance) {
            throw new Exception("Cannot remove {$amount}; current balance is {$currentBalance}");
        }
        
        $newBalance = $currentBalance - $amount;
        
        // Record as negative credit
        Capsule::table('tblcredit')
            ->insert([
                'clientid' => $clientId,
                'amount' => -$amount,
                'remaining' => $newBalance,
                'date' => date('Y-m-d H:i:s'),
                'description' => $description
            ]);
        
        Capsule::table('mod_credit_transactions')->insert([
            'client_id' => $clientId,
            'amount' => -$amount,
            'type' => 'remove',
            'description' => $description,
            'admin_id' => $adminId ?? $_SESSION['adminid']
        ]);
        
        logActivity("Credit of {$amount} removed from client #{$clientId}: {$description}");
        
        return ['success' => true, 'new_balance' => $newBalance];
    }
    
    public function applyCreditToInvoice($clientId, $invoiceId, $amount = null) {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->where('userid', $clientId)
            ->first();
        
        if (!$invoice) {
            throw new Exception('Invoice not found');
        }
        
        $creditBalance = $this->getBalance($clientId);
        if ($creditBalance <= 0) {
            throw new Exception('No credit available');
        }
        
        // Determine amount to apply
        $applyAmount = $amount ?? min($creditBalance, $invoice->total);
        
        // Apply credit
        Capsule::table('tblcredit')
            ->insert([
                'clientid' => $clientId,
                'amount' => -$applyAmount,
                'remaining' => $creditBalance - $applyAmount,
                'date' => date('Y-m-d H:i:s'),
                'description' => "Applied to Invoice #{$invoiceId}"
            ]);
        
        // Update invoice
        $newBalance = $invoice->balance - $applyAmount;
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update(['balance' => $newBalance]);
        
        // Record transaction
        Capsule::table('mod_credit_transactions')->insert([
            'client_id' => $clientId,
            'amount' => -$applyAmount,
            'type' => 'apply',
            'reference' => $invoiceId,
            'description' => "Applied to Invoice #{$invoiceId}"
        ]);
        
        // Log transaction
        logActivity("Credit of {$applyAmount} applied to invoice #{$invoiceId} for client #{$clientId}");
        logTransaction('Credit', ['amount' => $applyAmount], 'Applied');
        
        return ['success' => true, 'applied' => $applyAmount, 'remaining_balance' => $creditBalance - $applyAmount];
    }
    
    public function getBalance($clientId) {
        $credits = Capsule::table('tblcredit')
            ->where('clientid', $clientId)
            ->sum('amount');
        
        return (float) ($credits ?? 0);
    }
    
    private function getSetting($key, $default = null) {
        return Capsule::table('tblconfiguration')
            ->where('setting', $key)
            ->value('value') ?? $default;
    }
}
```

### Step 3: Create Credit Add Hook

Allow credit additions from various sources:

```php
// File: /includes/hooks/credit_management.php

add_hook('InvoiceRefunded', 1, function($vars) {
    // Automatically add refund as credit to client account
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    $creditManager = new \WHMCS\Billing\CreditManager();
    
    $creditManager->addCredit(
        $invoice->userid,
        $vars['amount'],
        "Refund for Invoice #{$invoiceId}",
        null,
        $vars['transactionid']
    );
    
    sendTemplatedEmail('RefundAsCredit', $invoice->userid, [
        'amount' => $vars['amount'],
        'invoice_id' => $invoiceId
    ]);
});

add_hook('ServiceCancellation', 1, function($vars) {
    // Add credit for unused service time on cancellation
    $hostingId = $vars['hosting_id'];
    $hosting = Capsule::table('tblhosting')
        ->where('id', $hostingId)
        ->first();
    
    // Calculate unused days credit
    $daysRemaining = calculateUnusedDays($hostingId);
    if ($daysRemaining > 0) {
        $dailyRate = getDailyRate($hosting->packageid);
        $creditAmount = $dailyRate * $daysRemaining;
        
        $creditManager = new \WHMCS\Billing\CreditManager();
        $creditManager->addCredit(
            $hosting->userid,
            $creditAmount,
            "Unused time credit on service cancellation - {$daysRemaining} days",
            null,
            $hostingId
        );
    }
});

add_hook('DailyCronJob', 1, function($vars) {
    // Auto-apply credits to invoices
    $autoApply = Capsule::table('tblconfiguration')
        ->where('setting', 'CreditAutoApply')
        ->value('value');
    
    if ($autoApply !== 'on') {
        return;
    }
    
    $threshold = Capsule::table('tblconfiguration')
        ->where('setting', 'CreditAutoApplyThreshold')
        ->value('value') ?? 10;
    
    // Find invoices with outstanding balance
    $invoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('balance', '>', 0)
        ->where('balance', '<=', $threshold)
        ->get();
    
    $creditManager = new \WHMCS\Billing\CreditManager();
    
    foreach ($invoices as $invoice) {
        $balance = $creditManager->getBalance($invoice->userid);
        
        if ($balance > 0) {
            $creditManager->applyCreditToInvoice($invoice->userid, $invoice->id);
            logActivity("Auto-applied credit to invoice #{$invoice->id}");
        }
    }
});
```

### Step 4: Create Credit Management Admin Interface

Add admin area functionality:

```php
// File: /modules/addons/credit_manager/admin.php

function credit_manager_admin($vars) {
    $action = $_GET['action'] ?? 'overview';
    
    switch ($action) {
        case 'add_credit':
            return handleAddCredit();
        case 'remove_credit':
            return handleRemoveCredit();
        case 'apply_credit':
            return handleApplyCredit();
        case 'view_transactions':
            return viewTransactions($_GET['client_id']);
        case 'overview':
        default:
            return creditManagerOverview();
    }
}

function handleAddCredit() {
    check_token('WHMCS.admin.default');
    
    $clientId = (int) $_POST['client_id'];
    $amount = (float) $_POST['amount'];
    $description = sanitize($_POST['description']);
    
    $creditManager = new \WHMCS\Billing\CreditManager();
    
    try {
        $result = $creditManager->addCredit($clientId, $amount, $description);
        flashMessage("Credit of {$amount} added. New balance: {$result['new_balance']}", 'success');
    } catch (Exception $e) {
        flashMessage($e->getMessage(), 'error');
    }
    
    return redirect('credits.php?action=view_transactions&client_id=' . $clientId);
}

function creditManagerOverview() {
    $stats = [
        'total_credits' => Capsule::table('tblcredit')->sum('amount'),
        'credit_transactions_today' => Capsule::table('mod_credit_transactions')
            ->whereDate('created_at', date('Y-m-d'))
            ->count(),
        'pending_applications' => Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->whereRaw('balance > 0')
            ->count(),
        'top_credit_balances' => Capsule::table('tblcredit')
            ->selectRaw('clientid, SUM(amount) as total')
            ->groupBy('clientid')
            ->orderBy('total', 'desc')
            ->limit(10)
            ->get()
    ];
    
    return [
        'template' => 'admin/credit_overview',
        'vars' => ['stats' => $stats]
    ];
}
```

### Step 5: Credit Reporting

Generate credit reports:

```php
// File: /includes/hooks/credit_reporting.php

add_hook('DailyCronJob', 1, function($vars) {
    // Generate daily credit report
    $today = date('Y-m-d');
    $yesterday = date('Y-m-d', strtotime('-1 day'));
    
    $report = [
        'date' => $today,
        'new_credits' => Capsule::table('mod_credit_transactions')
            ->whereDate('created_at', $yesterday)
            ->where('type', 'add')
            ->sum('amount'),
        'credits_used' => abs(Capsule::table('mod_credit_transactions')
            ->whereDate('created_at', $yesterday)
            ->where('type', 'apply')
            ->sum('amount')),
        'credits_removed' => abs(Capsule::table('mod_credit_transactions')
            ->whereDate('created_at', $yesterday)
            ->where('type', 'remove')
            ->sum('amount')),
        'net_change' => Capsule::table('mod_credit_transactions')
            ->whereDate('created_at', $yesterday)
            ->sum('amount'),
        'by_type' => Capsule::table('mod_credit_transactions')
            ->whereDate('created_at', $yesterday)
            ->groupBy('type')
            ->selectRaw('type, COUNT(*) as count, SUM(amount) as total')
            ->get()
    ];
    
    // Save report
    file_put_contents(
        __DIR__ . "/../storage/logs/credit_reports/{$yesterday}.json",
        json_encode($report, JSON_PRETTY_PRINT)
    );
    
    // Alert admin on large credits
    $largeCredits = Capsule::table('mod_credit_transactions')
        ->whereDate('created_at', $yesterday)
        ->where('type', 'add')
        ->where('amount', '>', 100)
        ->count();
    
    if ($largeCredits > 0) {
        sendAdminNotification(
            'email',
            'Credit Alert',
            "{$largeCredits} credits over $100 were added yesterday."
        );
    }
});

function getClientCreditHistory($clientId) {
    return Capsule::table('mod_credit_transactions')
        ->where('client_id', $clientId)
        ->orderBy('created_at', 'desc')
        ->get();
}
```

## Verification Checklist

- [ ] Credit configuration settings saved correctly
- [ ] Add credit function working
- [ ] Remove credit function working
- [ ] Apply credit to invoice working
- [ ] Balance calculation accurate
- [ ] Auto-apply credits functioning
- [ ] Admin interface accessible
- [ ] Credit transactions logged correctly
- [ ] Email notifications sent
- [ ] Reports generated accurately

## Related Skills and Documentation

- [WHMCS Invoice Automation](whmcs-invoice-automation-workflow.md)
- [WHMCS Refund Workflow](whmcs-refund-workflow.md)
- [WHMCS Billing Audit](whmcs-billing-audit-workflow.md)
- WHMCS Documentation: Credit Management
- WHMCS Documentation: Client Credits

## Notes

- Document credit policies clearly for clients and staff
- Set approval thresholds for large credit transactions
- Review credit balances regularly for accuracy
- Keep detailed audit trail for compliance
- Consider credit expiration policies
- Monitor credit usage patterns for fraud detection
- Reconcile credit balances with accounting records