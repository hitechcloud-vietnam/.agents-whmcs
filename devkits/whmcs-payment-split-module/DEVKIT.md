# WHMCS Payment Split Module DevKit

## Header

**Purpose:** Payment split module that enables distributing incoming payments across multiple accounts, products, or affiliates based on configurable rules.

**Module Type:** Payment/Accounting Module

**Use Case:** Hosting companies needing to split payments between multiple revenue streams, pay affiliates, or distribute subscription revenue across different products.

---

## Complete Code Template

### File Structure
```
whmcs-payment-split-module/
├── README.md
├── DEVKIT.md
├── payment_split.php        # Main split logic
├── hooks.php               # WHMCS hook integrations
├── split_rules.php         # Rule management
└── templates/
    └── admin_split.tpl
```

### Main Module File: payment_split.php

```php
<?php
/**
 * WHMCS Payment Split Module
 * 
 * Provides payment distribution across multiple recipients.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('PAYMENT_SPLIT_VERSION', '1.0.0');

// Split Types
define('SPLIT_FIXED', 'fixed');
define('SPLIT_PERCENTAGE', 'percentage');
define('SPLIT_RATIO', 'ratio');

// Split Status
define('SPLIT_STATUS_PENDING', 'pending');
define('SPLIT_STATUS_COMPLETED', 'completed');
define('SPLIT_STATUS_FAILED', 'failed');
define('SPLIT_STATUS_REVERSED', 'reversed');

/**
 * Process payment split
 */
function payment_split_process($paymentId, $invoiceId) {
    $payment = payment_get($paymentId);
    
    if (!$payment) {
        return ['success' => false, 'error' => 'Payment not found'];
    }
    
    $invoice = get_invoice($invoiceId);
    
    // Get applicable split rules
    $rules = payment_split_get_rules($invoice, $payment);
    
    if (empty($rules)) {
        // No rules, record full amount to primary account
        return payment_split_record_single($paymentId, $invoiceId);
    }
    
    // Calculate splits
    $splits = payment_split_calculate($payment['amount'], $rules);
    
    // Execute splits
    $results = payment_split_execute($paymentId, $invoiceId, $splits);
    
    return $results;
}

/**
 * Get payment details
 */
function payment_get($paymentId) {
    $query = "SELECT * FROM tblaccounts WHERE id = ?";
    $result = full_query($query, [$paymentId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get applicable split rules
 */
function payment_split_get_rules($invoice, $payment) {
    $rules = [];
    
    // Check product-specific rules
    $items = invoice_get_items($invoice['id']);
    foreach ($items as $item) {
        $itemRules = payment_split_get_product_rules($item['relid'], $item['type']);
        $rules = array_merge($rules, $itemRules);
    }
    
    // Check client-specific rules
    $clientRules = payment_split_get_client_rules($invoice['userid']);
    $rules = array_merge($rules, $clientRules);
    
    // Check invoice-specific rules
    $invoiceRules = payment_split_get_invoice_rules($invoice['id']);
    $rules = array_merge($rules, $invoiceRules);
    
    // Remove duplicates and prioritize
    return payment_split_deduplicate($rules);
}

/**
 * Get product-specific rules
 */
function payment_split_get_product_rules($productId, $type) {
    $query = "SELECT * FROM mod_payment_split_rules 
              WHERE rule_type = 'product' 
              AND (target_id = ? OR target_id = 0)
              AND target_type = ?
              AND status = 'active'";
    $result = full_query($query, [$productId, $type]);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = $row;
    }
    
    return $rules;
}

/**
 * Get client-specific rules
 */
function payment_split_get_client_rules($clientId) {
    $query = "SELECT * FROM mod_payment_split_rules 
              WHERE rule_type = 'client' 
              AND target_id = ?
              AND status = 'active'";
    $result = full_query($query, [$clientId]);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = $row;
    }
    
    return $rules;
}

/**
 * Get invoice-specific rules
 */
function payment_split_get_invoice_rules($invoiceId) {
    $query = "SELECT * FROM mod_payment_split_rules 
              WHERE rule_type = 'invoice' 
              AND target_id = ?
              AND status = 'active'";
    $result = full_query($query, [$invoiceId]);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = $row;
    }
    
    return $rules;
}

/**
 * Deduplicate and prioritize rules
 */
function payment_split_deduplicate($rules) {
    $seen = [];
    $deduplicated = [];
    
    foreach ($rules as $rule) {
        $key = $rule['recipient_id'] . '-' . $rule['split_type'];
        
        if (isset($seen[$key])) {
            // Keep higher priority rule
            if ($rule['priority'] > $seen[$key]['priority']) {
                $deduplicated = array_filter($deduplicated, 
                    fn($r) => $r['recipient_id'] != $rule['recipient_id']);
                $deduplicated[] = $rule;
                $seen[$key] = $rule;
            }
        } else {
            $deduplicated[] = $rule;
            $seen[$key] = $rule;
        }
    }
    
    return $deduplicated;
}

/**
 * Calculate payment splits
 */
function payment_split_calculate($totalAmount, $rules) {
    $splits = [];
    $remaining = $totalAmount;
    
    // Sort rules by priority
    usort($rules, fn($a, $b) => $b['priority'] - $a['priority']);
    
    foreach ($rules as $index => $rule) {
        if ($remaining <= 0) break;
        
        $isLast = ($index == count($rules) - 1);
        
        switch ($rule['split_type']) {
            case SPLIT_PERCENTAGE:
                $amount = $isLast ? $remaining : round($totalAmount * ($rule['split_value'] / 100), 2);
                break;
                
            case SPLIT_FIXED:
                $amount = min($rule['split_value'], $remaining);
                break;
                
            case SPLIT_RATIO:
                $ratio = $rule['split_value'];
                $totalRatio = array_sum(array_column($rules, 'split_value'));
                $amount = $isLast ? $remaining : round($totalAmount * ($ratio / $totalRatio), 2);
                break;
        }
        
        $splits[] = [
            'recipient_id' => $rule['recipient_id'],
            'recipient_type' => $rule['recipient_type'],
            'amount' => $amount,
            'rule_id' => $rule['id']
        ];
        
        $remaining -= $amount;
    }
    
    return $splits;
}

/**
 * Execute payment splits
 */
function payment_split_execute($paymentId, $invoiceId, $splits) {
    $results = [];
    $totalDistributed = 0;
    
    foreach ($splits as $split) {
        $result = payment_split_record($paymentId, $invoiceId, $split);
        
        if ($result['success']) {
            $totalDistributed += $split['amount'];
        }
        
        $results[] = $result;
        
        // Process affiliate commission if applicable
        if ($split['recipient_type'] == 'affiliate') {
            payment_split_process_affiliate($split);
        }
    }
    
    return [
        'success' => true,
        'splits' => $results,
        'total_distributed' => $totalDistributed
    ];
}

/**
 * Record single payment split
 */
function payment_split_record($paymentId, $invoiceId, $split) {
    $data = [
        'payment_id' => $paymentId,
        'invoice_id' => $invoiceId,
        'recipient_id' => $split['recipient_id'],
        'recipient_type' => $split['recipient_type'],
        'amount' => $split['amount'],
        'rule_id' => $split['rule_id'] ?? null,
        'status' => SPLIT_STATUS_COMPLETED,
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_splits', $data);
    
    // Update recipient balance based on type
    payment_split_update_balance($split['recipient_type'], $split['recipient_id'], $split['amount']);
    
    return [
        'success' => true,
        'recipient_id' => $split['recipient_id'],
        'amount' => $split['amount']
    ];
}

/**
 * Record single payment (no split)
 */
function payment_split_record_single($paymentId, $invoiceId) {
    $payment = payment_get($paymentId);
    
    $data = [
        'payment_id' => $paymentId,
        'invoice_id' => $invoiceId,
        'recipient_id' => 0,
        'recipient_type' => 'primary',
        'amount' => $payment['amount'],
        'status' => SPLIT_STATUS_COMPLETED,
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_splits', $data);
    
    return ['success' => true, 'single' => true];
}

/**
 * Update recipient balance
 */
function payment_split_update_balance($type, $recipientId, $amount) {
    switch ($type) {
        case 'affiliate':
            $table = 'tblaffiliates';
            $balanceField = 'balance';
            break;
        case 'vendor':
            $table = 'mod_payment_vendors';
            $balanceField = 'balance';
            break;
        case 'account':
            $table = 'mod_payment_accounts';
            $balanceField = 'balance';
            break;
        default:
            return;
    }
    
    full_query("UPDATE {$table} SET {$balanceField} = {$balanceField} + ? WHERE id = ?", 
               [$amount, $recipientId]);
}

/**
 * Process affiliate commission
 */
function payment_split_process_affiliate($split) {
    $query = "SELECT * FROM tblaffiliates WHERE id = ?";
    $result = full_query($query, [$split['recipient_id']]);
    $affiliate = mysql_fetch_assoc($result);
    
    if ($affiliate) {
        // Log commission
        insert_query('tblaffiliatelogs', [
            'affiliateid' => $split['recipient_id'],
            'referralid' => $affiliate['clientid'],
            'amount' => $split['amount'],
            'percentage' => $affiliate['commission'],
            'date' => date('Y-m-d'),
            'invoiceid' => 0
        ]);
    }
}

/**
 * Create split rule
 */
function payment_split_create_rule($data) {
    $fields = [
        'name', 'rule_type', 'target_type', 'target_id',
        'recipient_type', 'recipient_id', 'split_type', 'split_value',
        'priority', 'conditions', 'status', 'created_at'
    ];
    
    $values = [
        $data['name'], $data['rule_type'], $data['target_type'] ?? 'product',
        $data['target_id'] ?? 0, $data['recipient_type'], $data['recipient_id'],
        $data['split_type'], $data['split_value'], $data['priority'] ?? 50,
        json_encode($data['conditions'] ?? []), 'active', date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_split_rules', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Update split rule
 */
function payment_split_update_rule($ruleId, $data) {
    $allowedFields = ['name', 'split_type', 'split_value', 'priority', 'status'];
    $updateData = [];
    
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            $updateData[$field] = $data[$field];
        }
    }
    
    $updateData['updated_at'] = date('Y-m-d H:i:s');
    update_query('mod_payment_split_rules', $updateData, ['id' => $ruleId]);
}

/**
 * Delete split rule
 */
function payment_split_delete_rule($ruleId) {
    update_query('mod_payment_split_rules', ['status' => 'inactive'], ['id' => $ruleId]);
}

/**
 * Get all split rules
 */
function payment_split_get_all_rules($filters = []) {
    $where = "status = 'active'";
    $params = [];
    
    if (!empty($filters['rule_type'])) {
        $where .= " AND rule_type = ?";
        $params[] = $filters['rule_type'];
    }
    
    if (!empty($filters['recipient_type'])) {
        $where .= " AND recipient_type = ?";
        $params[] = $filters['recipient_type'];
    }
    
    $query = "SELECT r.*, 
              CASE r.recipient_type 
                WHEN 'affiliate' THEN a.name
                WHEN 'vendor' THEN v.name
                WHEN 'account' THEN a.name
              END as recipient_name
              FROM mod_payment_split_rules r
              LEFT JOIN tblaffiliates a ON r.recipient_type = 'affiliate' AND r.recipient_id = a.id
              LEFT JOIN mod_payment_vendors v ON r.recipient_type = 'vendor' AND r.recipient_id = v.id
              LEFT JOIN mod_payment_accounts a ON r.recipient_type = 'account' AND r.recipient_id = a.id
              WHERE {$where}
              ORDER BY r.priority DESC";
    
    $result = full_query($query, $params);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['conditions'] = json_decode($row['conditions'], true);
        $rules[] = $row;
    }
    
    return $rules;
}

/**
 * Reverse payment split
 */
function payment_split_reverse($paymentId) {
    $query = "SELECT * FROM mod_payment_splits WHERE payment_id = ?";
    $result = full_query($query, [$paymentId]);
    
    while ($split = mysql_fetch_assoc($result)) {
        // Update balance (reverse)
        payment_split_update_balance(
            $split['recipient_type'], 
            $split['recipient_id'], 
            -$split['amount']
        );
        
        // Mark as reversed
        update_query('mod_payment_splits', 
                     ['status' => SPLIT_STATUS_REVERSED], 
                     ['id' => $split['id']]);
    }
    
    return ['success' => true];
}

/**
 * Get split history for payment
 */
function payment_split_get_history($paymentId) {
    $query = "SELECT ps.*, r.name as rule_name
              FROM mod_payment_splits ps
              LEFT JOIN mod_payment_split_rules r ON ps.rule_id = r.id
              WHERE ps.payment_id = ?
              ORDER BY ps.created_at DESC";
    $result = full_query($query, [$paymentId]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $history[] = $row;
    }
    
    return $history;
}

/**
 * Get recipient balance
 */
function payment_split_get_balance($recipientType, $recipientId) {
    switch ($recipientType) {
        case 'affiliate':
            $query = "SELECT balance FROM tblaffiliates WHERE id = ?";
            break;
        case 'vendor':
            $query = "SELECT balance FROM mod_payment_vendors WHERE id = ?";
            break;
        case 'account':
            $query = "SELECT balance FROM mod_payment_accounts WHERE id = ?";
            break;
        default:
            return 0;
    }
    
    $result = full_query($query, [$recipientId]);
    $data = mysql_fetch_assoc($result);
    
    return $data['balance'] ?? 0;
}

/**
 * Generate split report
 */
function payment_split_generate_report($startDate, $endDate, $recipientType = null) {
    $where = "ps.created_at BETWEEN ? AND ?";
    $params = [$startDate, $endDate];
    
    if ($recipientType) {
        $where .= " AND ps.recipient_type = ?";
        $params[] = $recipientType;
    }
    
    $query = "SELECT 
                ps.recipient_type, ps.recipient_id, r.name as rule_name,
                COUNT(*) as transaction_count,
                SUM(ps.amount) as total_amount,
                p.name as recipient_name
              FROM mod_payment_splits ps
              LEFT JOIN mod_payment_split_rules r ON ps.rule_id = r.id
              LEFT JOIN tblaffiliates a ON ps.recipient_type = 'affiliate' AND ps.recipient_id = a.id
              LEFT JOIN mod_payment_vendors v ON ps.recipient_type = 'vendor' AND ps.recipient_id = v.id
              LEFT JOIN mod_payment_accounts acc ON ps.recipient_type = 'account' AND ps.recipient_id = acc.id
              COALESCE(a.name, v.name, acc.name, 'Primary') as pname
              WHERE {$where}
              GROUP BY ps.recipient_type, ps.recipient_id
              ORDER BY total_amount DESC";
    
    $result = full_query($query, $params);
    
    $report = [];
    while ($row = mysql_fetch_assoc($result)) {
        $report[] = $row;
    }
    
    return $report;
}

/**
 * Validate split rule
 */
function payment_split_validate_rule($data, &$errors) {
    if (empty($data['name'])) {
        $errors[] = "Rule name is required";
    }
    
    if (empty($data['recipient_type'])) {
        $errors[] = "Recipient type is required";
    }
    
    if (empty($data['recipient_id'])) {
        $errors[] = "Recipient is required";
    }
    
    if (empty($data['split_type'])) {
        $errors[] = "Split type is required";
    }
    
    if (!isset($data['split_value']) || $data['split_value'] <= 0) {
        $errors[] = "Valid split value is required";
    }
    
    if ($data['split_type'] == SPLIT_PERCENTAGE && $data['split_value'] > 100) {
        $errors[] = "Percentage cannot exceed 100%";
    }
    
    return count($errors) === 0;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Payment Split Module Hooks
 */

// Hook: Process payment split on payment received
add_hook('PaymentReceived', 1, function($params) {
    $paymentId = $params['paymentId'];
    $invoiceId = $params['invoiceId'];
    
    $result = payment_split_process($paymentId, $invoiceId);
    
    if ($result['success']) {
        logActivity("Payment split processed for payment #{$paymentId}: " . 
                   count($result['splits']) . " recipients");
    }
    
    return $result;
});

// Hook: Calculate split preview before payment
add_hook('InvoicePaid', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $paymentId = $params['paymentId'] ?? 0;
    
    $invoice = get_invoice($invoiceId);
    $rules = payment_split_get_rules($invoice, ['amount' => $invoice['total']]);
    
    if (!empty($rules)) {
        $splits = payment_split_calculate($invoice['total'], $rules);
        
        return [
            'split_preview' => $splits,
            'split_count' => count($splits)
        ];
    }
});

// Hook: Reverse split on refund
add_hook('RefundPayment', 1, function($params) {
    $paymentId = $params['paymentId'];
    
    payment_split_reverse($paymentId);
    
    return ['success' => true, 'action' => 'split_reversed'];
});

// Hook: Add split info to transaction display
add_hook('AdminAreaViewTransaction', 1, function($params) {
    $paymentId = $params['paymentId'];
    $splits = payment_split_get_history($paymentId);
    
    return [
        'payment_splits' => $splits
    ];
});

// Hook: Pre-split validation
add_hook('BeforePaymentProcess', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    $invoice = get_invoice($invoiceId);
    
    // Check for blocking rules
    $blockingRules = payment_split_get_blocking_rules($invoice);
    
    if (!empty($blockingRules)) {
        return [
            'blocked' => true,
            'reason' => $blockingRules[0]['reason']
        ];
    }
});

// Hook: Handle affiliate signup bonus
add_hook('AffiliateSignup', 1, function($params) {
    $affiliateId = $params['affiliateId'];
    $clientId = $params['clientId'];
    
    // Check for signup bonus split rule
    $query = "SELECT * FROM mod_payment_split_rules 
              WHERE rule_type = 'affiliate_signup' 
              AND recipient_id = ?
              AND status = 'active'";
    $result = full_query($query, [$affiliateId]);
    
    if ($rule = mysql_fetch_assoc($result)) {
        $bonus = $rule['split_value'];
        
        // Credit to affiliate
        update_query('tblaffiliates', ['balance' => $bonus], ['id' => $affiliateId]);
        
        logActivity("Signup bonus of {$bonus} credited to affiliate #{$affiliateId}");
    }
});

// Hook: Apply default split for new products
add_hook('ProductCreated', 1, function($params) {
    $productId = $params['productId'];
    
    // Check for default split rules
    $query = "SELECT * FROM mod_payment_split_rules 
              WHERE rule_type = 'default' 
              AND target_type = 'product'
              AND target_id = 0
              AND status = 'active'";
    $result = full_query($query);
    
    while ($rule = mysql_fetch_assoc($result)) {
        // Create copy for new product
        $data = $rule;
        unset($data['id']);
        $data['target_id'] = $productId;
        payment_split_create_rule($data);
    }
});

// Hook: Record split on partial payment
add_hook('InvoicePaymentApplied', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    $amount = $params['amount'];
    
    // For partial payments, calculate proportional splits
    $invoice = get_invoice($invoiceId);
    $rules = payment_split_get_rules($invoice, ['amount' => $amount]);
    
    if (!empty($rules)) {
        $splits = payment_split_calculate($amount, $rules);
        
        return [
            'partial_split' => true,
            'splits' => $splits
        ];
    }
});
```

---

## Database Schema

```sql
-- Payment split rules
CREATE TABLE `mod_payment_split_rules` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `rule_type` ENUM('product', 'client', 'invoice', 'affiliate', 'default') DEFAULT 'product',
    `target_type` VARCHAR(50) DEFAULT 'product',
    `target_id` INT DEFAULT 0,
    `recipient_type` ENUM('affiliate', 'vendor', 'account', 'primary') NOT NULL,
    `recipient_id` INT NOT NULL,
    `split_type` ENUM('fixed', 'percentage', 'ratio') DEFAULT 'percentage',
    `split_value` DECIMAL(10,4) NOT NULL,
    `priority` INT DEFAULT 50,
    `conditions` JSON DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_rule_type` (`rule_type`),
    INDEX `idx_target` (`target_type`, `target_id`),
    INDEX `idx_recipient` (`recipient_type`, `recipient_id`)
);

-- Payment splits record
CREATE TABLE `mod_payment_splits` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `payment_id` INT NOT NULL,
    `invoice_id` INT NOT NULL,
    `recipient_id` INT NOT NULL,
    `recipient_type` VARCHAR(50) NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `rule_id` INT DEFAULT NULL,
    `status` ENUM('pending', 'completed', 'failed', 'reversed') DEFAULT 'completed',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_payment` (`payment_id`),
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_recipient` (`recipient_type`, `recipient_id`),
    FOREIGN KEY (`payment_id`) REFERENCES `tblaccounts`(`id`) ON DELETE CASCADE
);

-- Payment vendors (for vendor splits)
CREATE TABLE `mod_payment_vendors` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `email` VARCHAR(255) DEFAULT NULL,
    `bank_details` TEXT DEFAULT NULL,
    `balance` DECIMAL(10,2) DEFAULT 0,
    `payout_threshold` DECIMAL(10,2) DEFAULT 0,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Payment accounts (for internal account splits)
CREATE TABLE `mod_payment_accounts` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `account_number` VARCHAR(100) DEFAULT NULL,
    `balance` DECIMAL(10,2) DEFAULT 0,
    `description` TEXT DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `PaymentReceived` | 1 | Process payment split |
| `InvoicePaid` | 1 | Preview split calculation |
| `RefundPayment` | 1 | Reverse split on refund |
| `AdminAreaViewTransaction` | 1 | Show split details |
| `BeforePaymentProcess` | 1 | Validate splits |
| `AffiliateSignup` | 1 | Handle affiliate bonuses |
| `ProductCreated` | 1 | Apply default rules |
| `InvoicePaymentApplied` | 1 | Handle partial payments |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement core split logic
- [ ] Create rule management system
- [ ] Implement percentage split
- [ ] Implement fixed amount split
- [ ] Implement ratio-based split
- [ ] Add affiliate integration
- [ ] Create vendor management
- [ ] Add account management
- [ ] Build admin interface
- [ ] Implement split preview
- [ ] Add split reversal
- [ ] Create reporting system
- [ ] Add balance tracking
- [ ] Test various split scenarios
- [ ] Verify affiliate commissions
- [ ] Test partial payments
- [ ] Add split validation
- [ ] Implement split history