# WHMCS Adjustment Module DevKit

## Header

**Purpose:** Invoice adjustment module that handles credit memos, manual adjustments, service credits, and invoice corrections with full audit trail.

**Module Type:** Billing/Operations Module

**Use Case:** Hosting companies needing to apply credits, correct billing errors, create credit memos, or make administrative adjustments to invoices and accounts.

---

## Complete Code Template

### File Structure
```
whmcs-adjustment-module/
├── README.md
├── DEVKIT.md
├── adjustment.php          # Main adjustment logic
├── hooks.php               # WHMCS hook integrations
└── templates/
    └── admin_adjustment.tpl
```

### Main Module File: adjustment.php

```php
<?php
/**
 * WHMCS Adjustment Module
 * 
 * Handles invoice adjustments, credits, and corrections.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('ADJUSTMENT_MODULE_VERSION', '1.0.0');

// Adjustment Types
define('ADJ_CREDIT_MEMO', 'credit_memo');
define('ADJ_SERVICE_CREDIT', 'service_credit');
define('ADJ_BILLING_CORRECTION', 'billing_correction');
define('ADJ_DISCOUNT', 'discount');
define('ADJ_REFUND', 'refund');
define('ADJ_WRITE_OFF', 'write_off');
define('ADJ_MANUAL', 'manual');

// Adjustment Status
define('ADJ_STATUS_PENDING', 'pending');
define('ADJ_STATUS_APPROVED', 'approved');
define('ADJ_STATUS_APPLIED', 'applied');
define('ADJ_STATUS_REJECTED', 'rejected');
define('ADJ_STATUS_VOID', 'void');

// Adjustment Actions
define('ADJ_ACTION_ADD', 'add');
define('ADJ_ACTION_DEDUCT', 'deduct');
define('ADJ_ACTION_REVERSE', 'reverse');

/**
 * Create adjustment
 */
function adjustment_create($data) {
    $adjId = adjustment_generate_id();
    
    $fields = [
        'adjustment_id', 'invoice_id', 'client_id', 'type', 'action',
        'amount', 'description', 'reason', 'reference',
        'requires_approval', 'status', 'created_by', 'approved_by',
        'applied_by', 'created_at', 'updated_at'
    ];
    
    $values = [
        $adjId, $data['invoice_id'] ?? null, $data['client_id'],
        $data['type'], $data['action'] ?? ADJ_ACTION_DEDUCT,
        $data['amount'], $data['description'], $data['reason'] ?? '',
        $data['reference'] ?? '', $data['requires_approval'] ?? false,
        ADJ_STATUS_PENDING, $data['created_by'] ?? $_SESSION['adminid'] ?? 0,
        null, null, date('Y-m-d H:i:s'), date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_adjustments', array_combine($fields, $values));
    
    $dbId = mysql_insert_id();
    
    // Log creation
    adjustment_log($dbId, 'created', [
        'type' => $data['type'],
        'amount' => $data['amount']
    ]);
    
    // Notify if approval required
    if ($data['requires_approval']) {
        adjustment_notify_approvers($dbId);
    }
    
    return [
        'success' => true,
        'adjustment_id' => $adjId,
        'db_id' => $dbId
    ];
}

/**
 * Generate unique adjustment ID
 */
function adjustment_generate_id() {
    return 'ADJ-' . date('ymd') . '-' . strtoupper(substr(md5(uniqid()), 0, 6));
}

/**
 * Approve adjustment
 */
function adjustment_approve($adjId, $approvedBy, $notes = '') {
    $adj = adjustment_get($adjId);
    
    if (!$adj) {
        return ['success' => false, 'error' => 'Adjustment not found'];
    }
    
    if ($adj['status'] != ADJ_STATUS_PENDING) {
        return ['success' => false, 'error' => 'Adjustment not pending'];
    }
    
    update_query('mod_adjustments', [
        'status' => ADJ_STATUS_APPROVED,
        'approved_by' => $approvedBy,
        'approved_at' => date('Y-m-d H:i:s'),
        'admin_notes' => $notes,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $adjId]);
    
    adjustment_log($adjId, 'approved', [
        'approved_by' => $approvedBy,
        'notes' => $notes
    ]);
    
    return ['success' => true];
}

/**
 * Reject adjustment
 */
function adjustment_reject($adjId, $rejectedBy, $reason) {
    $adj = adjustment_get($adjId);
    
    if (!$adj || $adj['status'] != ADJ_STATUS_PENDING) {
        return ['success' => false, 'error' => 'Invalid adjustment'];
    }
    
    update_query('mod_adjustments', [
        'status' => ADJ_STATUS_REJECTED,
        'approved_by' => $rejectedBy,
        'approved_at' => date('Y-m-d H:i:s'),
        'admin_notes' => $reason,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $adjId]);
    
    adjustment_log($adjId, 'rejected', [
        'rejected_by' => $rejectedBy,
        'reason' => $reason
    ]);
    
    return ['success' => true];
}

/**
 * Apply adjustment to invoice
 */
function adjustment_apply($adjId, $appliedBy = null) {
    $adj = adjustment_get($adjId);
    
    if (!$adj) {
        return ['success' => false, 'error' => 'Adjustment not found'];
    }
    
    if (!in_array($adj['status'], [ADJ_STATUS_APPROVED, ADJ_STATUS_PENDING])) {
        return ['success' => false, 'error' => 'Adjustment cannot be applied'];
    }
    
    // Determine action
    $amount = $adj['amount'];
    $description = $adj['description'];
    
    if ($adj['action'] == ADJ_ACTION_DEDUCT) {
        $amount = -abs($amount);
    } elseif ($adj['action'] == ADJ_ACTION_ADD) {
        $amount = abs($amount);
    }
    
    // Apply to invoice if exists
    if ($adj['invoice_id']) {
        adjustment_apply_to_invoice($adj);
    }
    
    // Record as accounting entry
    adjustment_record_transaction($adj, $amount);
    
    // Update status
    update_query('mod_adjustments', [
        'status' => ADJ_STATUS_APPLIED,
        'applied_by' => $appliedBy ?? $_SESSION['adminid'] ?? 0,
        'applied_at' => date('Y-m-d H:i:s'),
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $adjId]);
    
    adjustment_log($adjId, 'applied', [
        'amount' => $amount,
        'applied_by' => $appliedBy
    ]);
    
    return [
        'success' => true,
        'amount' => $amount,
        'applied_to_invoice' => $adj['invoice_id']
    ];
}

/**
 * Apply adjustment to invoice
 */
function adjustment_apply_to_invoice($adj) {
    $invoiceId = $adj['invoice_id'];
    $amount = $adj['amount'];
    
    if ($adj['action'] == ADJ_ACTION_DEDUCT) {
        $amount = -abs($amount);
    }
    
    // Add line item to invoice
    $itemData = [
        'invoiceid' => $invoiceId,
        'userid' => $adj['client_id'],
        'description' => $adj['description'],
        'amount' => $amount,
        'taxed' => 0
    ];
    
    insert_query('tblinvoiceitems', $itemData);
    
    // Update invoice total
    if ($amount > 0) {
        full_query("UPDATE tblinvoices SET total = total + ? WHERE id = ?", [$amount, $invoiceId]);
    } else {
        full_query("UPDATE tblinvoices SET total = total + ? WHERE id = ?", [abs($amount), $invoiceId]);
    }
    
    // Check if fully paid
    adjustment_check_invoice_paid($invoiceId);
}

/**
 * Check if invoice is now paid
 */
function adjustment_check_invoice_paid($invoiceId) {
    $query = "SELECT i.total, COALESCE(SUM(a.amount), 0) as paid
              FROM tblinvoices i
              LEFT JOIN tblaccounts a ON i.id = a.invoiceid
              WHERE i.id = ?";
    $result = full_query($query, [$invoiceId]);
    $data = mysql_fetch_assoc($result);
    
    if ($data['paid'] >= $data['total']) {
        update_query('tblinvoices', [
            'status' => 'Paid',
            'datepaid' => date('Y-m-d H:i:s')
        ], ['id' => $invoiceId]);
    }
}

/**
 * Record adjustment transaction
 */
function adjustment_record_transaction($adj, $amount) {
    $description = $adj['description'];
    if ($adj['reference']) {
        $description .= ' (Ref: ' . $adj['reference'] . ')';
    }
    
    insert_query('tblaccounts', [
        'userid' => $adj['client_id'],
        'description' => $description,
        'amount' => $amount,
        'date' => date('Y-m-d'),
        'invoiceid' => $adj['invoice_id'],
        'transid' => $adj['adjustment_id']
    ]);
}

/**
 * Get adjustment
 */
function adjustment_get($adjId) {
    $query = "SELECT a.*, c.firstname, c.lastname, c.email,
              i.invoicenum, i.total as invoice_total
              FROM mod_adjustments a
              JOIN tblclients c ON a.client_id = c.id
              LEFT JOIN tblinvoices i ON a.invoice_id = i.id
              WHERE a.id = ? OR a.adjustment_id = ?";
    $result = full_query($query, [$adjId, $adjId]);
    return mysql_fetch_assoc($result);
}

/**
 * Create credit memo
 */
function adjustment_create_credit_memo($clientId, $amount, $description, $reason = '') {
    return adjustment_create([
        'client_id' => $clientId,
        'type' => ADJ_CREDIT_MEMO,
        'action' => ADJ_ACTION_ADD,
        'amount' => $amount,
        'description' => $description,
        'reason' => $reason,
        'requires_approval' => false
    ]);
}

/**
 * Create service credit
 */
function adjustment_create_service_credit($clientId, $invoiceId, $amount, $description, $reason = '') {
    return adjustment_create([
        'client_id' => $clientId,
        'invoice_id' => $invoiceId,
        'type' => ADJ_SERVICE_CREDIT,
        'action' => ADJ_ACTION_DEDUCT,
        'amount' => $amount,
        'description' => $description,
        'reason' => $reason,
        'requires_approval' => false
    ]);
}

/**
 * Create billing correction
 */
function adjustment_create_correction($clientId, $invoiceId, $amount, $description, $reason, $requiresApproval = true) {
    return adjustment_create([
        'client_id' => $clientId,
        'invoice_id' => $invoiceId,
        'type' => ADJ_BILLING_CORRECTION,
        'action' => $amount > 0 ? ADJ_ACTION_ADD : ADJ_ACTION_DEDUCT,
        'amount' => abs($amount),
        'description' => $description,
        'reason' => $reason,
        'requires_approval' => $requiresApproval
    ]);
}

/**
 * Create discount adjustment
 */
function adjustment_create_discount($clientId, $invoiceId, $amount, $description, $reason = '') {
    return adjustment_create([
        'client_id' => $clientId,
        'invoice_id' => $invoiceId,
        'type' => ADJ_DISCOUNT,
        'action' => ADJ_ACTION_DEDUCT,
        'amount' => $amount,
        'description' => $description,
        'reason' => $reason,
        'requires_approval' => false
    ]);
}

/**
 * Write off bad debt
 */
function adjustment_write_off($clientId, $invoiceId, $amount, $reason, $writeOffAccount = 'bad_debt') {
    return adjustment_create([
        'client_id' => $clientId,
        'invoice_id' => $invoiceId,
        'type' => ADJ_WRITE_OFF,
        'action' => ADJ_ACTION_DEDUCT,
        'amount' => $amount,
        'description' => 'Bad debt write-off: ' . $reason,
        'reason' => $reason,
        'reference' => $writeOffAccount,
        'requires_approval' => true
    ]);
}

/**
 * Void adjustment
 */
function adjustment_void($adjId, $voidedBy, $reason = '') {
    $adj = adjustment_get($adjId);
    
    if (!$adj) {
        return ['success' => false, 'error' => 'Adjustment not found'];
    }
    
    if ($adj['status'] == ADJ_STATUS_VOID || $adj['status'] == ADJ_STATUS_REJECTED) {
        return ['success' => false, 'error' => 'Adjustment already voided'];
    }
    
    // Reverse transaction if applied
    if ($adj['status'] == ADJ_STATUS_APPLIED) {
        adjustment_reverse($adjId);
    }
    
    update_query('mod_adjustments', [
        'status' => ADJ_STATUS_VOID,
        'admin_notes' => $reason,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $adjId]);
    
    adjustment_log($adjId, 'voided', [
        'voided_by' => $voidedBy,
        'reason' => $reason
    ]);
    
    return ['success' => true];
}

/**
 * Reverse applied adjustment
 */
function adjustment_reverse($adjId) {
    $adj = adjustment_get($adjId);
    
    if (!$adj || $adj['status'] != ADJ_STATUS_APPLIED) {
        return ['success' => false, 'error' => 'Cannot reverse'];
    }
    
    $amount = $adj['amount'];
    if ($adj['action'] == ADJ_ACTION_DEDUCT) {
        $amount = -abs($amount);
    } else {
        $amount = abs($amount);
    }
    
    // Create reversal transaction
    insert_query('tblaccounts', [
        'userid' => $adj['client_id'],
        'description' => 'REVERSAL: ' . $adj['description'],
        'amount' => -$amount,
        'date' => date('Y-m-d'),
        'invoiceid' => $adj['invoice_id'],
        'transid' => $adj['adjustment_id'] . '-RV'
    ]);
    
    // Update adjustment status
    update_query('mod_adjustments', [
        'status' => ADJ_STATUS_PENDING,
        'applied_by' => null,
        'applied_at' => null,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $adjId]);
    
    adjustment_log($adjId, 'reversed');
    
    return ['success' => true];
}

/**
 * Get client adjustments
 */
function adjustment_get_client($clientId, $limit = 100) {
    $query = "SELECT a.*, i.invoicenum
              FROM mod_adjustments a
              LEFT JOIN tblinvoices i ON a.invoice_id = i.id
              WHERE a.client_id = ?
              ORDER BY a.created_at DESC LIMIT ?";
    $result = full_query($query, [$clientId, $limit]);
    
    $adjustments = [];
    while ($row = mysql_fetch_assoc($result)) {
        $adjustments[] = $row;
    }
    
    return $adjustments;
}

/**
 * Get pending adjustments
 */
function adjustment_get_pending($filters = []) {
    $where = "a.status = ?";
    $params = [ADJ_STATUS_PENDING];
    
    if (!empty($filters['type'])) {
        $where .= " AND a.type = ?";
        $params[] = $filters['type'];
    }
    
    if (!empty($filters['client_id'])) {
        $where .= " AND a.client_id = ?";
        $params[] = $filters['client_id'];
    }
    
    if (!empty($filters['min_amount'])) {
        $where .= " AND a.amount >= ?";
        $params[] = $filters['min_amount'];
    }
    
    $query = "SELECT a.*, c.firstname, c.lastname, i.invoicenum
              FROM mod_adjustments a
              JOIN tblclients c ON a.client_id = c.id
              LEFT JOIN tblinvoices i ON a.invoice_id = i.id
              WHERE {$where}
              ORDER BY a.created_at ASC";
    
    $result = full_query($query, $params);
    
    $adjustments = [];
    while ($row = mysql_fetch_assoc($result)) {
        $adjustments[] = $row;
    }
    
    return $adjustments;
}

/**
 * Log adjustment action
 */
function adjustment_log($adjId, $action, $data = []) {
    if (is_numeric($adjId)) {
        $query = "SELECT id FROM mod_adjustments WHERE id = ?";
    } else {
        $query = "SELECT id FROM mod_adjustments WHERE adjustment_id = ?";
    }
    
    $result = full_query($query, [$adjId]);
    $adj = mysql_fetch_assoc($result);
    
    insert_query('mod_adjustment_logs', [
        'adjustment_id' => $adj['id'],
        'action' => $action,
        'data' => json_encode($data),
        'created_by' => $_SESSION['adminid'] ?? 0,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

/**
 * Notify approvers
 */
function adjustment_notify_approvers($adjId) {
    $adj = adjustment_get($adjId);
    
    // Get users with adjustment permissions
    $query = "SELECT email FROM tbladmins 
              WHERE privileges LIKE '%adjustments%' OR role = 'owner'";
    $result = full_query($query);
    
    while ($admin = mysql_fetch_assoc($result)) {
        send_email($admin['email'], 'Adjustment Approval Required', [
            'adjustment_id' => $adj['adjustment_id'],
            'client' => $adj['firstname'] . ' ' . $adj['lastname'],
            'amount' => formatCurrency($adj['amount']),
            'description' => $adj['description'],
            'type' => $adj['type']
        ]);
    }
}

/**
 * Get adjustment logs
 */
function adjustment_get_logs($adjId) {
    $query = "SELECT l.*, a.username as admin_name
              FROM mod_adjustment_logs l
              LEFT JOIN tbladmins a ON l.created_by = a.id
              WHERE l.adjustment_id = ?
              ORDER BY l.created_at DESC";
    $result = full_query($query, [$adjId]);
    
    $logs = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['data'] = json_decode($row['data'], true);
        $logs[] = $row;
    }
    
    return $logs;
}

/**
 * Get adjustment statistics
 */
function adjustment_get_stats($startDate = null, $endDate = null) {
    $where = "1=1";
    $params = [];
    
    if ($startDate && $endDate) {
        $where .= " AND created_at BETWEEN ? AND ?";
        $params = [$startDate, $endDate];
    }
    
    $query = "SELECT 
                COUNT(*) as total,
                SUM(CASE WHEN status = 'applied' THEN amount ELSE 0 END) as total_applied,
                SUM(CASE WHEN type = 'credit_memo' THEN 1 ELSE 0 END) as credit_memos,
                SUM(CASE WHEN type = 'service_credit' THEN 1 ELSE 0 END) as service_credits,
                SUM(CASE WHEN type = 'billing_correction' THEN 1 ELSE 0 END) as corrections,
                SUM(CASE WHEN type = 'discount' THEN 1 ELSE 0 END) as discounts,
                SUM(CASE WHEN type = 'write_off' THEN 1 ELSE 0 END) as write_offs,
                SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) as pending,
                SUM(CASE WHEN status = 'rejected' THEN 1 ELSE 0 END) as rejected
              FROM mod_adjustments
              WHERE {$where}";
    
    $result = full_query($query, $params);
    return mysql_fetch_assoc($result);
}

/**
 * Validate adjustment
 */
function adjustment_validate($data, &$errors = []) {
    if (empty($data['client_id'])) {
        $errors[] = 'Client is required';
    }
    
    if (!isset($data['amount']) || $data['amount'] <= 0) {
        $errors[] = 'Amount must be positive';
    }
    
    if (empty($data['type'])) {
        $errors[] = 'Adjustment type is required';
    }
    
    if (empty($data['description'])) {
        $errors[] = 'Description is required';
    }
    
    return count($errors) === 0;
}

/**
 * Bulk create adjustments
 */
function adjustment_bulk_create($clientIds, $data) {
    $created = 0;
    
    foreach ($clientIds as $clientId) {
        $data['client_id'] = $clientId;
        $result = adjustment_create($data);
        
        if ($result['success']) {
            $created++;
            
            // Auto-apply if not requiring approval
            if (!$data['requires_approval']) {
                adjustment_apply($result['db_id']);
            }
        }
    }
    
    return ['created' => $created];
}

/**
 * Create credit memo for refund
 */
function adjustment_credit_memo_from_refund($clientId, $invoiceId, $amount, $originalPaymentId) {
    return adjustment_create([
        'client_id' => $clientId,
        'invoice_id' => $invoiceId,
        'type' => ADJ_CREDIT_MEMO,
        'action' => ADJ_ACTION_ADD,
        'amount' => $amount,
        'description' => 'Credit memo for refund',
        'reason' => 'Refund processed',
        'reference' => 'REF-' . $originalPaymentId,
        'requires_approval' => false
    ]);
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Adjustment Module Hooks
 */

// Hook: Create adjustment for service credit
add_hook('ServiceCreditRequest', 1, function($params) {
    $clientId = $params['clientId'];
    $invoiceId = $params['invoiceId'] ?? null;
    $amount = $params['amount'];
    $description = $params['description'];
    
    $result = adjustment_create_service_credit($clientId, $invoiceId, $amount, $description, 'Service credit request');
    
    if ($result['success']) {
        // Auto-apply small credits
        if ($amount <= 10) {
            adjustment_apply($result['db_id']);
        }
    }
    
    return $result;
});

// Hook: Handle billing correction on invoice
add_hook('InvoiceCorrection', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    $amount = $params['amount'];
    $reason = $params['reason'];
    
    $invoice = get_query_vals('tblinvoices', 'userid', ['id' => $invoiceId]);
    
    return adjustment_create_correction($invoice['userid'], $invoiceId, $amount, 
        'Billing correction', $reason, true);
});

// Hook: Create discount adjustment
add_hook('InvoiceDiscountRequest', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    $amount = $params['amount'];
    $reason = $params['reason'];
    
    $invoice = get_query_vals('tblinvoices', 'userid', ['id' => $invoiceId]);
    
    return adjustment_create_discount($invoice['userid'], $invoiceId, $amount,
        'Discount applied: ' . $reason);
});

// Hook: Track adjustment in invoice view
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    $query = "SELECT * FROM mod_adjustments WHERE invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    $adjustments = [];
    while ($adj = mysql_fetch_assoc($result)) {
        $adjustments[] = $adj;
    }
    
    return ['adjustments' => $adjustments];
});

// Hook: Auto-apply credits on invoice view
add_hook('AdminAreaPageHook', 1, function($params) {
    if ($params['filename'] == 'invoices' && $_POST['action'] == 'apply_credit') {
        $adjId = $_POST['adjustment_id'];
        
        return adjustment_apply($adjId);
    }
});

// Hook: Create credit memo on refund
add_hook('RefundProcessed', 1, function($params) {
    $clientId = $params['clientId'];
    $invoiceId = $params['invoiceId'];
    $amount = $params['amount'];
    $paymentId = $params['paymentId'];
    
    // Create credit memo
    $result = adjustment_credit_memo_from_refund($clientId, $invoiceId, $amount, $paymentId);
    
    if ($result['success']) {
        adjustment_apply($result['db_id']);
    }
});

// Hook: Process bulk discounts
add_hook('BulkDiscountAction', 1, function($params) {
    $invoiceIds = $params['invoice_ids'];
    $amount = $params['amount'];
    $reason = $params['reason'];
    
    $created = 0;
    foreach ($invoiceIds as $invoiceId) {
        $invoice = get_query_vals('tblinvoices', 'userid', ['id' => $invoiceId]);
        
        $result = adjustment_create_discount($invoice['userid'], $invoiceId, $amount, 
            'Bulk discount: ' . $reason);
        
        if ($result['success']) {
            adjustment_apply($result['db_id']);
            $created++;
        }
    }
    
    return ['created' => $created];
});

// Hook: Add adjustment to client summary
add_hook('ClientSummaryPage', 1, function($params) {
    $clientId = $params['clientId'];
    
    $stats = adjustment_get_stats();
    $recent = adjustment_get_client($clientId, 5);
    
    return [
        'adjustment_stats' => $stats,
        'recent_adjustments' => $recent
    ];
});

// Hook: Prevent service changes with pending adjustments
add_hook('BeforeServiceChange', 1, function($params) {
    $clientId = $params['userId'];
    
    $pending = adjustment_get_pending(['client_id' => $clientId]);
    
    if (!empty($pending)) {
        return [
            'warning' => true,
            'message' => 'You have pending adjustments requiring approval'
        ];
    }
});
```

---

## Database Schema

```sql
-- Adjustments
CREATE TABLE `mod_adjustments` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `adjustment_id` VARCHAR(50) NOT NULL UNIQUE,
    `invoice_id` INT DEFAULT NULL,
    `client_id` INT NOT NULL,
    `type` ENUM('credit_memo', 'service_credit', 'billing_correction', 'discount', 'refund', 'write_off', 'manual') NOT NULL,
    `action` ENUM('add', 'deduct', 'reverse') DEFAULT 'deduct',
    `amount` DECIMAL(10,2) NOT NULL,
    `description` TEXT NOT NULL,
    `reason` TEXT DEFAULT NULL,
    `reference` VARCHAR(255) DEFAULT NULL,
    `requires_approval` TINYINT(1) DEFAULT 0,
    `status` ENUM('pending', 'approved', 'applied', 'rejected', 'void') DEFAULT 'pending',
    `created_by` INT NOT NULL,
    `approved_by` INT DEFAULT NULL,
    `approved_at` DATETIME DEFAULT NULL,
    `applied_by` INT DEFAULT NULL,
    `applied_at` DATETIME DEFAULT NULL,
    `admin_notes` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_client` (`client_id`),
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_type` (`type`),
    INDEX `idx_status` (`status`),
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Adjustment logs
CREATE TABLE `mod_adjustment_logs` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `adjustment_id` INT NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `created_by` INT DEFAULT 0,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_adjustment` (`adjustment_id`),
    FOREIGN KEY (`adjustment_id`) REFERENCES `mod_adjustments`(`id`) ON DELETE CASCADE
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ServiceCreditRequest` | 1 | Handle service credits |
| `InvoiceCorrection` | 1 | Create billing corrections |
| `InvoiceDiscountRequest` | 1 | Apply discounts |
| `AdminAreaViewInvoice` | 1 | Show adjustments |
| `AdminAreaPageHook` | 1 | Apply adjustments |
| `RefundProcessed` | 1 | Create credit memos |
| `BulkDiscountAction` | 1 | Bulk discounts |
| `ClientSummaryPage` | 1 | Add to client summary |
| `BeforeServiceChange` | 1 | Check pending adjustments |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement adjustment creation
- [ ] Create approval workflow
- [ ] Add application logic
- [ ] Implement voiding
- [ ] Create reversal functionality
- [ ] Build credit memo handling
- [ ] Add service credits
- [ ] Create billing corrections
- [ ] Build discounts
- [ ] Add write-offs
- [ ] Create bulk operations
- [ ] Build admin interface
- [ ] Add logging
- [ ] Implement notifications
- [ ] Create reports
- [ ] Test approval workflow
- [ ] Verify accounting entries
- [ ] Add validation