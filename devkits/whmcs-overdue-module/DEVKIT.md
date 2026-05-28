# WHMCS Overdue Module DevKit

## Header

**Purpose:** Overdue payment handling module that manages late payment notifications, automated follow-ups, service suspension logic, and dunning campaigns.

**Module Type:** Billing/Collections Module

**Use Case:** Hosting companies needing automated late payment reminders, progressive dunning sequences, configurable grace periods, and automated service suspension for overdue accounts.

---

## Complete Code Template

### File Structure
```
whmcs-overdue-module/
├── README.md
├── DEVKIT.md
├── overdue.php           # Main overdue logic
├── hooks.php            # WHMCS hook integrations
├── dunning.php          # Dunning sequences
└── templates/
    └── admin_overdue.tpl
```

### Main Module File: overdue.php

```php
<?php
/**
 * WHMCS Overdue Module
 * 
 * Handles overdue payment processing and dunning.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('OVERDUE_MODULE_VERSION', '1.0.0');

// Overdue Status
define('OVERDUE_STATUS_CURRENT', 'current');
define('OVERDUE_STATUS_OVERDUE', 'overdue');
define('OVERDUE_STATUS_FINAL', 'final');
define('OVERDUE_STATUS_SUSPENDED', 'suspended');
define('OVERDUE_STATUS_TERMINATED', 'terminated');

// Dunning Stage
define('DUNNING_FIRST_REMINDER', 'first_reminder');
define('DUNNING_SECOND_REMINDER', 'second_reminder');
define('DUNNING_FINAL_NOTICE', 'final_notice');
define('DUNNING_SUSPENSION_WARNING', 'suspension_warning');
define('DUNNING_TERMINATION_WARNING', 'termination_warning');

/**
 * Check if invoice is overdue
 */
function overdue_check($invoiceId) {
    $invoice = overdue_get_invoice($invoiceId);
    
    if (!$invoice) {
        return ['is_overdue' => false];
    }
    
    if ($invoice['status'] == 'Paid' || $invoice['status'] == 'Cancelled') {
        return ['is_overdue' => false, 'status' => $invoice['status']];
    }
    
    $dueDate = strtotime($invoice['duedate']);
    $now = time();
    $daysOverdue = ceil(($now - $dueDate) / 86400);
    
    return [
        'is_overdue' => $daysOverdue > 0,
        'days_overdue' => max(0, $daysOverdue),
        'invoice' => $invoice
    ];
}

/**
 * Get invoice details
 */
function overdue_get_invoice($invoiceId) {
    $query = "SELECT i.*, c.id as client_id, c.firstname, c.lastname, c.email,
              c.groupid, SUM(a.amount) as amount_paid
              FROM tblinvoices i
              JOIN tblclients c ON i.userid = c.id
              LEFT JOIN tblaccounts a ON i.id = a.invoiceid
              WHERE i.id = ?";
    $result = full_query($query, [$invoiceId]);
    return mysql_fetch_assoc($result);
}

/**
 * Process overdue invoice
 */
function overdue_process($invoiceId) {
    $check = overdue_check($invoiceId);
    
    if (!$check['is_overdue']) {
        return ['success' => false, 'reason' => 'Invoice not overdue'];
    }
    
    $invoice = $check['invoice'];
    $daysOverdue = $check['days_overdue'];
    
    // Determine dunning stage
    $stage = overdue_determine_stage($daysOverdue);
    
    // Check if we've already sent this stage
    if (overdue_already_sent($invoiceId, $stage)) {
        return ['success' => false, 'reason' => 'Stage already sent'];
    }
    
    // Send appropriate notification
    $result = overdue_send_notification($invoice, $stage, $daysOverdue);
    
    if ($result['success']) {
        // Record the action
        overdue_record_action($invoiceId, $stage, $daysOverdue);
        
        // Check for suspension
        if (overdue_should_suspend($invoiceId, $stage)) {
            overdue_suspend_services($invoiceId);
        }
        
        // Check for termination
        if (overdue_should_terminate($invoiceId, $stage)) {
            overdue_terminate_services($invoiceId);
        }
    }
    
    return $result;
}

/**
 * Determine dunning stage based on days overdue
 */
function overdue_determine_stage($daysOverdue) {
    // Get configuration
    $config = overdue_get_config();
    
    if ($daysOverdue >= $config['termination_days']) {
        return DUNNING_TERMINATION_WARNING;
    }
    
    if ($daysOverdue >= $config['suspension_days']) {
        return DUNNING_SUSPENSION_WARNING;
    }
    
    if ($daysOverdue >= $config['final_notice_days']) {
        return DUNNING_FINAL_NOTICE;
    }
    
    if ($daysOverdue >= $config['second_reminder_days']) {
        return DUNNING_SECOND_REMINDER;
    }
    
    return DUNNING_FIRST_REMINDER;
}

/**
 * Get module configuration
 */
function overdue_get_config() {
    $defaults = [
        'first_reminder_days' => 1,
        'second_reminder_days' => 7,
        'final_notice_days' => 14,
        'suspension_days' => 21,
        'termination_days' => 30,
        'auto_suspend' => true,
        'auto_terminate' => false,
        'suspend_services' => true,
        'terminate_services' => false,
        'add_late_fee' => false,
        'late_fee_amount' => 0,
        'late_fee_percent' => 0
    ];
    
    $query = "SELECT `key`, `value` FROM mod_overdue_config";
    $result = full_query($query);
    
    $config = $defaults;
    while ($row = mysql_fetch_assoc($result)) {
        if (isset($defaults[$row['key']])) {
            $config[$row['key']] = is_numeric($row['value']) ? (float)$row['value'] : $row['value'];
        }
    }
    
    return $config;
}

/**
 * Check if stage already sent
 */
function overdue_already_sent($invoiceId, $stage) {
    $query = "SELECT id FROM mod_overdue_actions 
              WHERE invoice_id = ? AND stage = ? AND created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)";
    $result = full_query($query, [$invoiceId, $stage]);
    return (bool)mysql_fetch_assoc($result);
}

/**
 * Send overdue notification
 */
function overdue_send_notification($invoice, $stage, $daysOverdue) {
    $emailTemplate = overdue_get_email_template($stage);
    
    if (!$emailTemplate) {
        return ['success' => false, 'error' => 'Email template not found'];
    }
    
    // Prepare template data
    $templateData = overdue_prepare_template_data($invoice, $stage, $daysOverdue);
    
    // Send email
    $result = send_email(
        $invoice['email'],
        $emailTemplate['subject'],
        $templateData,
        $emailTemplate['id']
    );
    
    return [
        'success' => $result,
        'stage' => $stage,
        'days_overdue' => $daysOverdue,
        'template' => $emailTemplate['name']
    ];
}

/**
 * Get email template for stage
 */
function overdue_get_email_template($stage) {
    $templates = [
        DUNNING_FIRST_REMINDER => 'Overdue Invoice Reminder',
        DUNNING_SECOND_REMINDER => 'Overdue Invoice Second Reminder',
        DUNNING_FINAL_NOTICE => 'Invoice Final Notice',
        DUNNING_SUSPENSION_WARNING => 'Service Suspension Warning',
        DUNNING_TERMINATION_WARNING => 'Service Termination Warning'
    ];
    
    $templateName = $templates[$stage] ?? 'Overdue Invoice Reminder';
    
    $query = "SELECT * FROM tblemailtemplates WHERE name = ?";
    $result = full_query($query, [$templateName]);
    
    return mysql_fetch_assoc($result);
}

/**
 * Prepare email template data
 */
function overdue_prepare_template_data($invoice, $stage, $daysOverdue) {
    $client = overdue_get_client($invoice['userid']);
    
    return [
        'client_first_name' => $client['firstname'],
        'client_last_name' => $client['lastname'],
        'invoice_number' => $invoice['invoicenum'] ?: $invoice['id'],
        'invoice_id' => $invoice['id'],
        'invoice_amount' => formatCurrency($invoice['total']),
        'invoice_balance' => formatCurrency($invoice['total'] - ($invoice['amount_paid'] ?? 0)),
        'invoice_date' => from UnixDate($invoice['date']),
        'invoice_due_date' => from UnixDate($invoice['duedate']),
        'days_overdue' => $daysOverdue,
        'payment_link' => overdue_get_payment_link($invoice['id']),
        'auto_suspend_date' => overdue_get_suspend_date(),
        'auto_terminate_date' => overdue_get_terminate_date()
    ];
}

/**
 * Get client details
 */
function overdue_get_client($clientId) {
    $query = "SELECT * FROM tblclients WHERE id = ?";
    $result = full_query($query, [$clientId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get payment link
 */
function overdue_get_payment_link($invoiceId) {
    return $GLOBALS['CONFIG']['SystemURL'] . '/viewinvoice.php?id=' . $invoiceId;
}

/**
 * Get suspend warning date
 */
function overdue_get_suspend_date() {
    $config = overdue_get_config();
    $suspendDays = $config['suspension_days'];
    return date('Y-m-d', strtotime('+' . $suspendDays . ' days'));
}

/**
 * Get terminate warning date
 */
function overdue_get_terminate_date() {
    $config = overdue_get_config();
    $terminateDays = $config['termination_days'];
    return date('Y-m-d', strtotime('+' . $terminateDays . ' days'));
}

/**
 * Record overdue action
 */
function overdue_record_action($invoiceId, $stage, $daysOverdue) {
    $data = [
        'invoice_id' => $invoiceId,
        'stage' => $stage,
        'days_overdue' => $daysOverdue,
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_overdue_actions', $data);
}

/**
 * Check if services should be suspended
 */
function overdue_should_suspend($invoiceId, $stage) {
    $config = overdue_get_config();
    
    return $config['auto_suspend'] && 
           ($stage == DUNNING_SUSPENSION_WARNING || $stage == DUNNING_TERMINATION_WARNING);
}

/**
 * Suspend overdue services
 */
function overdue_suspend_services($invoiceId) {
    $invoice = overdue_get_invoice($invoiceId);
    
    // Get services for this client
    $query = "SELECT id FROM tblhosting 
              WHERE userid = ? AND domainstatus = 'Active'";
    $result = full_query($query, [$invoice['userid']]);
    
    $suspended = [];
    while ($service = mysql_fetch_assoc($result)) {
        update_query('tblhosting', ['domainstatus' => 'Suspended'], ['id' => $service['id']]);
        $suspended[] = $service['id'];
        
        overdue_log_service_status($service['id'], 'suspended', $invoiceId);
    }
    
    // Update invoice status
    update_query('tblinvoices', ['status' => 'Overdue'], ['id' => $invoiceId]);
    
    return ['suspended_count' => count($suspended), 'service_ids' => $suspended];
}

/**
 * Check if services should be terminated
 */
function overdue_should_terminate($invoiceId, $stage) {
    $config = overdue_get_config();
    
    return $config['auto_terminate'] && $stage == DUNNING_TERMINATION_WARNING;
}

/**
 * Terminate overdue services
 */
function overdue_terminate_services($invoiceId) {
    $invoice = overdue_get_invoice($invoiceId);
    
    $query = "SELECT id FROM tblhosting 
              WHERE userid = ? AND domainstatus IN ('Active', 'Suspended')";
    $result = full_query($query, [$invoice['userid']]);
    
    $terminated = [];
    while ($service = mysql_fetch_assoc($result)) {
        update_query('tblhosting', ['domainstatus' => 'Terminated'], ['id' => $service['id']]);
        $terminated[] = $service['id'];
        
        overdue_log_service_status($service['id'], 'terminated', $invoiceId);
    }
    
    return ['terminated_count' => count($terminated), 'service_ids' => $terminated];
}

/**
 * Log service status change
 */
function overdue_log_service_status($serviceId, $status, $invoiceId) {
    insert_query('mod_overdue_service_logs', [
        'service_id' => $serviceId,
        'status' => $status,
        'invoice_id' => $invoiceId,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

/**
 * Reactivate suspended services on payment
 */
function overdue_reactivate_services($clientId) {
    // Check for any overdue invoices
    $hasOverdue = overdue_client_has_overdue($clientId);
    
    if ($hasOverdue) {
        return ['success' => false, 'reason' => 'Client has overdue invoices'];
    }
    
    // Reactivate suspended services
    $query = "SELECT id FROM tblhosting 
              WHERE userid = ? AND domainstatus = 'Suspended'";
    $result = full_query($query, [$clientId]);
    
    $reactivated = [];
    while ($service = mysql_fetch_assoc($result)) {
        update_query('tblhosting', ['domainstatus' => 'Active'], ['id' => $service['id']]);
        $reactivated[] = $service['id'];
        
        overdue_log_service_status($service['id'], 'reactivated', 0);
    }
    
    return [
        'success' => true,
        'reactivated_count' => count($reactivated),
        'service_ids' => $reactivated
    ];
}

/**
 * Check if client has overdue invoices
 */
function overdue_client_has_overdue($clientId) {
    $query = "SELECT COUNT(*) as count FROM tblinvoices 
              WHERE userid = ? AND status NOT IN ('Paid', 'Cancelled') 
              AND duedate < CURDATE()";
    $result = full_query($query, [$clientId]);
    $data = mysql_fetch_assoc($result);
    
    return $data['count'] > 0;
}

/**
 * Get overdue invoices for client
 */
function overdue_get_client_invoices($clientId) {
    $query = "SELECT i.*, DATEDIFF(CURDATE(), i.duedate) as days_overdue
              FROM tblinvoices i
              WHERE i.userid = ? AND i.status NOT IN ('Paid', 'Cancelled')
              AND i.duedate < CURDATE()
              ORDER BY i.duedate ASC";
    $result = full_query($query, [$clientId]);
    
    $invoices = [];
    while ($row = mysql_fetch_assoc($result)) {
        $invoices[] = $row;
    }
    
    return $invoices;
}

/**
 * Add late fee to overdue invoice
 */
function overdue_add_late_fee($invoiceId) {
    $config = overdue_get_config();
    
    if (!$config['add_late_fee']) {
        return ['success' => false, 'reason' => 'Late fees disabled'];
    }
    
    $invoice = overdue_get_invoice($invoiceId);
    $balance = $invoice['total'] - ($invoice['amount_paid'] ?? 0);
    
    $lateFee = 0;
    if ($config['late_fee_percent'] > 0) {
        $lateFee = $balance * ($config['late_fee_percent'] / 100);
    } else {
        $lateFee = $config['late_fee_amount'];
    }
    
    if ($lateFee <= 0) {
        return ['success' => false, 'reason' => 'Invalid late fee amount'];
    }
    
    // Add late fee item to invoice
    insert_query('tblinvoiceitems', [
        'invoiceid' => $invoiceId,
        'userid' => $invoice['userid'],
        'description' => 'Late payment fee',
        'amount' => $lateFee,
        'taxed' => 0
    ]);
    
    // Update invoice total
    full_query("UPDATE tblinvoices SET total = total + ? WHERE id = ?", [$lateFee, $invoiceId]);
    
    // Log the action
    overdue_log_action($invoiceId, 'late_fee_added', ['amount' => $lateFee]);
    
    return ['success' => true, 'late_fee' => $lateFee];
}

/**
 * Log generic action
 */
function overdue_log_action($invoiceId, $action, $data = []) {
    insert_query('mod_overdue_logs', [
        'invoice_id' => $invoiceId,
        'action' => $action,
        'data' => json_encode($data),
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

/**
 * Get overdue statistics
 */
function overdue_get_stats($startDate = null, $endDate = null) {
    $where = "1=1";
    $params = [];
    
    if ($startDate && $endDate) {
        $where .= " AND created_at BETWEEN ? AND ?";
        $params = [$startDate, $endDate];
    }
    
    $query = "SELECT 
                COUNT(DISTINCT invoice_id) as total_actions,
                COUNT(CASE WHEN stage = 'first_reminder' THEN 1 END) as first_reminders,
                COUNT(CASE WHEN stage = 'second_reminder' THEN 1 END) as second_reminders,
                COUNT(CASE WHEN stage = 'final_notice' THEN 1 END) as final_notices,
                COUNT(CASE WHEN stage = 'suspension_warning' THEN 1 END) as suspension_warnings,
                COUNT(CASE WHEN stage = 'termination_warning' THEN 1 END) as termination_warnings
              FROM mod_overdue_actions
              WHERE {$where}";
    
    $result = full_query($query, $params);
    return mysql_fetch_assoc($result);
}

/**
 * Process all overdue invoices (cron)
 */
function overdue_process_all() {
    // Get all unpaid, overdue invoices
    $query = "SELECT i.id FROM tblinvoices i
              WHERE i.status NOT IN ('Paid', 'Cancelled')
              AND i.duedate < CURDATE()";
    $result = full_query($query);
    
    $processed = 0;
    while ($invoice = mysql_fetch_assoc($result)) {
        $result = overdue_process($invoice['id']);
        if ($result['success']) {
            $processed++;
        }
    }
    
    return ['processed' => $processed];
}

/**
 * Get dunning timeline for invoice
 */
function overdue_get_timeline($invoiceId) {
    $config = overdue_get_config();
    
    $timeline = [
        [
            'stage' => DUNNING_FIRST_REMINDER,
            'days' => $config['first_reminder_days'],
            'description' => 'First reminder'
        ],
        [
            'stage' => DUNNING_SECOND_REMINDER,
            'days' => $config['second_reminder_days'],
            'description' => 'Second reminder'
        ],
        [
            'stage' => DUNNING_FINAL_NOTICE,
            'days' => $config['final_notice_days'],
            'description' => 'Final notice'
        ],
        [
            'stage' => DUNNING_SUSPENSION_WARNING,
            'days' => $config['suspension_days'],
            'description' => 'Suspension warning'
        ],
        [
            'stage' => DUNNING_TERMINATION_WARNING,
            'days' => $config['termination_days'],
            'description' => 'Termination warning'
        ]
    ];
    
    $invoice = overdue_get_invoice($invoiceId);
    $dueDate = strtotime($invoice['duedate']);
    
    foreach ($timeline as &$item) {
        $item['date'] = date('Y-m-d', $dueDate + ($item['days'] * 86400));
        
        // Check if sent
        $query = "SELECT id FROM mod_overdue_actions 
                  WHERE invoice_id = ? AND stage = ? LIMIT 1";
        $result = full_query($query, [$invoiceId, $item['stage']]);
        $item['sent'] = (bool)mysql_fetch_assoc($result);
    }
    
    return $timeline;
}

/**
 * Update dunning configuration
 */
function overdue_update_config($key, $value) {
    $query = "INSERT INTO mod_overdue_config (`key`, `value`) VALUES (?, ?)
              ON DUPLICATE KEY UPDATE `value` = VALUES(`value`)";
    full_query($query, [$key, $value]);
}

/**
 * Get overdue actions history
 */
function overdue_get_history($invoiceId) {
    $query = "SELECT * FROM mod_overdue_actions 
              WHERE invoice_id = ? ORDER BY created_at DESC";
    $result = full_query($query, [$invoiceId]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $history[] = $row;
    }
    
    return $history;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Overdue Module Hooks
 */

// Hook: Process overdue invoices daily
add_hook('DailyCronJob', 1, function($params) {
    $result = overdue_process_all();
    
    logActivity("Overdue processing: {$result['processed']} invoices processed");
    
    return $result;
});

// Hook: Check for late fees on overdue
add_hook('DailyCronJob', 2, function($params) {
    $config = overdue_get_config();
    
    if (!$config['add_late_fee']) {
        return [];
    }
    
    // Add late fees to invoices overdue by configured days
    $query = "SELECT id FROM tblinvoices 
              WHERE status NOT IN ('Paid', 'Cancelled')
              AND duedate <= DATE_SUB(CURDATE(), INTERVAL ? DAY)
              AND id NOT IN (SELECT invoice_id FROM mod_overdue_fees_added)";
    $result = full_query($query, [$config['late_fee_days'] ?? 7]);
    
    $added = 0;
    while ($invoice = mysql_fetch_assoc($result)) {
        $result = overdue_add_late_fee($invoice['id']);
        if ($result['success']) {
            $added++;
        }
    }
    
    return ['late_fees_added' => $added];
});

// Hook: Reactivate services on payment
add_hook('InvoicePaid', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $invoice = overdue_get_invoice($invoiceId);
    
    // Check if client has any remaining overdue invoices
    $hasOverdue = overdue_client_has_overdue($invoice['userid']);
    
    if (!$hasOverdue) {
        return overdue_reactivate_services($invoice['userid']);
    }
});

// Hook: Update status on payment
add_hook('InvoicePaid', 2, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Update overdue actions to mark as resolved
    update_query('mod_overdue_actions', 
                 ['resolved_at' => date('Y-m-d H:i:s')], 
                 ['invoice_id' => $invoiceId, 'resolved_at' => null]);
});

// Hook: Add overdue info to client area
add_hook('ClientAreaPage', 1, function($params) {
    if ($_SESSION['uid']) {
        $overdueInvoices = overdue_get_client_invoices($_SESSION['uid']);
        
        if (!empty($overdueInvoices)) {
            return [
                'has_overdue' => true,
                'overdue_count' => count($overdueInvoices),
                'overdue_total' => array_sum(array_column($overdueInvoices, 'total'))
            ];
        }
    }
});

// Hook: Prevent service changes if overdue
add_hook('BeforeServiceChange', 1, function($params) {
    $clientId = $params['userId'] ?? $_SESSION['uid'];
    
    if ($clientId && overdue_client_has_overdue($clientId)) {
        return [
            'allowed' => false,
            'reason' => 'Please settle overdue invoices before making changes'
        ];
    }
});

// Hook: Add overdue status to invoice display
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $check = overdue_check($invoiceId);
    
    return [
        'is_overdue' => $check['is_overdue'],
        'days_overdue' => $check['days_overdue'] ?? 0,
        'timeline' => overdue_get_timeline($invoiceId)
    ];
});

// Hook: Custom late fee calculation
add_hook('CalculateLateFee', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    $invoice = overdue_get_invoice($invoiceId);
    $balance = $invoice['total'] - ($invoice['amount_paid'] ?? 0);
    
    $config = overdue_get_config();
    
    $lateFee = 0;
    if ($config['late_fee_percent'] > 0) {
        $lateFee = $balance * ($config['late_fee_percent'] / 100);
    } else {
        $lateFee = $config['late_fee_amount'];
    }
    
    return ['late_fee' => $lateFee];
});
```

---

## Database Schema

```sql
-- Overdue actions log
CREATE TABLE `mod_overdue_actions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `stage` VARCHAR(50) NOT NULL,
    `days_overdue` INT DEFAULT 0,
    `sent_via` VARCHAR(20) DEFAULT 'email',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `resolved_at` DATETIME DEFAULT NULL,
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_stage` (`stage`),
    INDEX `idx_created` (`created_at`)
);

-- Overdue logs
CREATE TABLE `mod_overdue_logs` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_invoice` (`invoice_id`)
);

-- Service status change logs
CREATE TABLE `mod_overdue_service_logs` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `service_id` INT NOT NULL,
    `status` VARCHAR(20) NOT NULL,
    `invoice_id` INT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_service` (`service_id`)
);

-- Late fees added tracking
CREATE TABLE `mod_overdue_fees_added` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL UNIQUE,
    `amount` DECIMAL(10,2) NOT NULL,
    `added_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`invoice_id`) REFERENCES `tblinvoices`(`id`) ON DELETE CASCADE
);

-- Configuration
CREATE TABLE `mod_overdue_config` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `key` VARCHAR(100) NOT NULL UNIQUE,
    `value` TEXT DEFAULT NULL,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `DailyCronJob` | 1 | Process all overdue invoices |
| `DailyCronJob` | 2 | Add late fees |
| `InvoicePaid` | 1 | Reactivate suspended services |
| `InvoicePaid` | 2 | Mark actions as resolved |
| `ClientAreaPage` | 1 | Show overdue warning |
| `BeforeServiceChange` | 1 | Block changes if overdue |
| `AdminAreaViewInvoice` | 1 | Show overdue timeline |
| `CalculateLateFee` | 1 | Custom late fee calculation |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement overdue detection
- [ ] Create dunning stages
- [ ] Build notification system
- [ ] Implement service suspension
- [ ] Add termination handling
- [ ] Create late fee system
- [ ] Build reactivation logic
- [ ] Add statistics reporting
- [ ] Create timeline display
- [ ] Build admin interface
- [ ] Add configuration options
- [ ] Test dunning sequences
- [ ] Verify suspension workflow
- [ ] Test reactivation
- [ ] Add email templates
- [ ] Implement grace periods
- [ ] Create export functionality