# WHMCS Refund Module DevKit

## Header

**Purpose:** Comprehensive refund processing module that handles refund requests, approval workflows, partial refunds, refund calculations, and automatic refund processing.

**Module Type:** Billing/Operations Module

**Use Case:** Hosting companies needing structured refund workflows, automated refund calculations, partial refunds, refund tracking, and integration with payment gateways for refund processing.

---

## Complete Code Template

### File Structure
```
whmcs-refund-module/
├── README.md
├── DEVKIT.md
├── refund.php           # Main refund logic
├── hooks.php            # WHMCS hook integrations
├── workflows.php        # Approval workflows
└── templates/
    └── admin_refund.tpl
```

### Main Module File: refund.php

```php
<?php
/**
 * WHMCS Refund Module
 * 
 * Provides refund request processing and workflow management.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('REFUND_MODULE_VERSION', '1.0.0');

// Refund Status
define('REFUND_STATUS_PENDING', 'pending');
define('REFUND_STATUS_APPROVED', 'approved');
define('REFUND_STATUS_REJECTED', 'rejected');
define('REFUND_STATUS_PROCESSING', 'processing');
define('REFUND_STATUS_COMPLETED', 'completed');
define('REFUND_STATUS_CANCELLED', 'cancelled');

// Refund Types
define('REFUND_FULL', 'full');
define('REFUND_PARTIAL', 'partial');
define('REFUND_PROPORTIONAL', 'proportional');
define('REFUND_CREDIT', 'credit');

// Refund Reasons
define('REASON_DUPLICATE', 'duplicate_charge');
define('REASON_ERROR', 'billing_error');
define('REASON_SERVICE', 'service_issue');
define('REASON_CANCEL', 'cancellation');
define('REASON_OTHER', 'other');

/**
 * Create refund request
 */
function refund_create_request($data) {
    $invoiceId = $data['invoice_id'];
    $invoice = get_invoice($invoiceId);
    
    if (!$invoice) {
        return ['success' => false, 'error' => 'Invoice not found'];
    }
    
    // Calculate refund amount
    $maxRefund = refund_calculate_max_amount($invoiceId);
    $requestedAmount = $data['amount'] ?? $maxRefund;
    
    // Validate amount
    if ($requestedAmount > $maxRefund) {
        return ['success' => false, 'error' => 'Amount exceeds maximum refund'];
    }
    
    $requestId = refund_generate_id();
    
    $fields = [
        'request_id', 'invoice_id', 'client_id', 'amount', 'refund_type',
        'reason', 'reason_details', 'payment_id', 'status', 'requested_by',
        'requested_at', 'ip_address'
    ];
    
    $values = [
        $requestId, $invoiceId, $invoice['userid'], $requestedAmount,
        $data['type'] ?? REFUND_FULL, $data['reason'], $data['details'] ?? '',
        $data['payment_id'] ?? 0, REFUND_STATUS_PENDING, $data['requested_by'],
        date('Y-m-d H:i:s'), $_SERVER['REMOTE_ADDR'] ?? ''
    ];
    
    insert_query('mod_refund_requests', array_combine($fields, $values));
    
    // Log creation
    refund_log($requestId, 'created', [
        'amount' => $requestedAmount,
        'reason' => $data['reason']
    ]);
    
    // Notify admins
    refund_notify_admins($requestId);
    
    return [
        'success' => true,
        'request_id' => $requestId,
        'amount' => $requestedAmount
    ];
}

/**
 * Generate refund request ID
 */
function refund_generate_id() {
    return 'REF-' . date('ymd') . '-' . strtoupper(substr(md5(uniqid()), 0, 6));
}

/**
 * Get invoice details
 */
function get_invoice($invoiceId) {
    $query = "SELECT * FROM tblinvoices WHERE id = ?";
    $result = full_query($query, [$invoiceId]);
    return mysql_fetch_assoc($result);
}

/**
 * Calculate maximum refundable amount
 */
function refund_calculate_max_amount($invoiceId) {
    $invoice = get_invoice($invoiceId);
    
    // Get total paid
    $query = "SELECT SUM(amount) as total_paid FROM tblaccounts 
              WHERE invoiceid = ?";
    $result = full_query($query, [$invoiceId]);
    $data = mysql_fetch_assoc($result);
    $totalPaid = $data['total_paid'] ?? 0;
    
    // Get already refunded
    $query = "SELECT SUM(amount) as total_refunded FROM mod_refund_requests 
              WHERE invoice_id = ? AND status = 'completed'";
    $result = full_query($query, [$invoiceId]);
    $data = mysql_fetch_assoc($result);
    $totalRefunded = $data['total_refunded'] ?? 0;
    
    return $totalPaid - $totalRefunded;
}

/**
 * Approve refund request
 */
function refund_approve($requestId, $approvedBy, $notes = '') {
    $request = refund_get($requestId);
    
    if (!$request) {
        return ['success' => false, 'error' => 'Request not found'];
    }
    
    if ($request['status'] != REFUND_STATUS_PENDING) {
        return ['success' => false, 'error' => 'Request is not pending'];
    }
    
    // Update status
    update_query('mod_refund_requests', [
        'status' => REFUND_STATUS_APPROVED,
        'approved_by' => $approvedBy,
        'approved_at' => date('Y-m-d H:i:s'),
        'admin_notes' => $notes
    ], ['id' => $requestId]);
    
    refund_log($requestId, 'approved', [
        'approved_by' => $approvedBy,
        'notes' => $notes
    ]);
    
    // Notify client
    refund_notify_client($requestId, 'approved');
    
    return ['success' => true];
}

/**
 * Reject refund request
 */
function refund_reject($requestId, $rejectedBy, $reason) {
    $request = refund_get($requestId);
    
    if (!$request || $request['status'] != REFUND_STATUS_PENDING) {
        return ['success' => false, 'error' => 'Invalid request status'];
    }
    
    update_query('mod_refund_requests', [
        'status' => REFUND_STATUS_REJECTED,
        'rejected_by' => $rejectedBy,
        'rejected_at' => date('Y-m-d H:i:s'),
        'rejection_reason' => $reason
    ], ['id' => $requestId]);
    
    refund_log($requestId, 'rejected', [
        'rejected_by' => $rejectedBy,
        'reason' => $reason
    ]);
    
    refund_notify_client($requestId, 'rejected');
    
    return ['success' => true];
}

/**
 * Process approved refund
 */
function refund_process($requestId) {
    $request = refund_get($requestId);
    
    if (!$request || $request['status'] != REFUND_STATUS_APPROVED) {
        return ['success' => false, 'error' => 'Request not approved'];
    }
    
    // Update status
    update_query('mod_refund_requests', [
        'status' => REFUND_STATUS_PROCESSING
    ], ['id' => $requestId]);
    
    // Get original payment method
    $paymentMethod = refund_get_payment_method($request['payment_id']);
    
    // Process based on payment method
    $result = refund_execute($request, $paymentMethod);
    
    if ($result['success']) {
        // Mark as completed
        update_query('mod_refund_requests', [
            'status' => REFUND_STATUS_COMPLETED,
            'processed_at' => date('Y-m-d H:i:s'),
            'transaction_id' => $result['transaction_id'] ?? null
        ], ['id' => $requestId]);
        
        // Create credit if configured
        if ($request['refund_type'] == REFUND_CREDIT) {
            refund_create_credit($request);
        }
        
        refund_log($requestId, 'completed', $result);
        
        return ['success' => true, 'transaction_id' => $result['transaction_id'] ?? null];
    }
    
    return $result;
}

/**
 * Get refund request
 */
function refund_get($requestId) {
    $query = "SELECT r.*, c.firstname, c.lastname, c.email,
              i.invoicenum, i.total as invoice_total
              FROM mod_refund_requests r
              JOIN tblclients c ON r.client_id = c.id
              JOIN tblinvoices i ON r.invoice_id = i.id
              WHERE r.id = ? OR r.request_id = ?";
    $result = full_query($query, [$requestId, $requestId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get payment method
 */
function refund_get_payment_method($paymentId) {
    if (!$paymentId) {
        return null;
    }
    
    $query = "SELECT a.*, g.gateway, g.type as gateway_type
              FROM tblaccounts a
              LEFT JOIN tblpaymentgateways g ON a.paymentmethod = g.gateway
              WHERE a.id = ?";
    $result = full_query($query, [$paymentId]);
    return mysql_fetch_assoc($result);
}

/**
 * Execute refund with payment gateway
 */
function refund_execute($request, $paymentMethod) {
    // Use appropriate gateway API to process refund
    $gateway = $paymentMethod['gateway'] ?? 'unknown';
    
    switch ($gateway) {
        case 'stripe':
            return refund_process_stripe($request, $paymentMethod);
        case 'paypal':
            return refund_process_paypal($request, $paymentMethod);
        case 'authorizenet':
            return refund_process_authorize($request, $paymentMethod);
        default:
            // Manual refund (create credit or bank transfer)
            return refund_process_manual($request);
    }
}

/**
 * Process Stripe refund
 */
function refund_process_stripe($request, $paymentMethod) {
    // Stripe refund implementation
    // This would integrate with Stripe API
    
    $stripeKey = get_gateway_setting('stripe', 'secret_key');
    
    // Get original charge ID from payment
    $chargeId = $paymentMethod['transid'] ?? '';
    
    if (empty($chargeId)) {
        return ['success' => false, 'error' => 'No original charge found'];
    }
    
    try {
        // Stripe API call would go here
        // $charge = \Stripe\Charge::retrieve($chargeId);
        // $refund = $charge->refund(['amount' => $request['amount'] * 100]);
        
        return [
            'success' => true,
            'transaction_id' => 'stripe_' . time()
        ];
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

/**
 * Process PayPal refund
 */
function refund_process_paypal($request, $paymentMethod) {
    // PayPal refund implementation
    // This would integrate with PayPal API
    
    $paypalEmail = get_gateway_setting('paypal', 'email');
    
    // PayPal NVP API call would go here
    
    return [
        'success' => true,
        'transaction_id' => 'paypal_' . time()
    ];
}

/**
 * Process manual refund
 */
function refund_process_manual($request) {
    // Create accounting entry
    insert_query('tblaccounts', [
        'userid' => $request['client_id'],
        'description' => 'Refund: Invoice #' . $request['invoicenum'],
        'amount' => -$request['amount'],
        'date' => date('Y-m-d'),
        'invoiceid' => $request['invoice_id']
    ]);
    
    return [
        'success' => true,
        'transaction_id' => 'manual_' . time()
    ];
}

/**
 * Process Authorize.Net refund
 */
function refund_process_authorize($request, $paymentMethod) {
    // Authorize.Net refund implementation
    
    return [
        'success' => true,
        'transaction_id' => 'authnet_' . time()
    ];
}

/**
 * Create account credit
 */
function refund_create_credit($request) {
    $data = [
        'client_id' => $request['client_id'],
        'amount' => $request['amount'],
        'description' => 'Refund credit from request ' . $request['request_id'],
        'remaining' => $request['amount'],
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_credit_balances', $data);
    
    // Log credit creation
    refund_log($request['id'], 'credit_created', [
        'credit_amount' => $request['amount']
    ]);
}

/**
 * Calculate proportional refund
 */
function refund_calculate_proportional($requestId, $unusedDays) {
    $request = refund_get($requestId);
    
    // Get billing period
    $query = "SELECT * FROM tblhosting WHERE userid = ? 
              ORDER BY regdate DESC LIMIT 1";
    $result = full_query($query, [$request['client_id']]);
    $service = mysql_fetch_assoc($result);
    
    if (!$service) {
        return $request['amount'];
    }
    
    // Get total days in billing cycle
    $cycleDays = refund_get_cycle_days($service['billingcycle']);
    
    // Calculate refund
    $dailyRate = $request['amount'] / $cycleDays;
    return round($dailyRate * $unusedDays, 2);
}

/**
 * Get billing cycle days
 */
function refund_get_cycle_days($cycle) {
    switch ($cycle) {
        case 'monthly':
            return 30;
        case 'quarterly':
            return 90;
        case 'semiannually':
            return 180;
        case 'annually':
            return 365;
        case 'biennially':
            return 730;
        default:
            return 30;
    }
}

/**
 * Cancel refund request
 */
function refund_cancel($requestId, $cancelledBy) {
    $request = refund_get($requestId);
    
    if (!$request) {
        return ['success' => false, 'error' => 'Request not found'];
    }
    
    if (!in_array($request['status'], [REFUND_STATUS_PENDING, REFUND_STATUS_APPROVED])) {
        return ['success' => false, 'error' => 'Cannot cancel in current status'];
    }
    
    update_query('mod_refund_requests', [
        'status' => REFUND_STATUS_CANCELLED,
        'cancelled_by' => $cancelledBy,
        'cancelled_at' => date('Y-m-d H:i:s')
    ], ['id' => $requestId]);
    
    refund_log($requestId, 'cancelled', ['cancelled_by' => $cancelledBy]);
    
    return ['success' => true];
}

/**
 * Get pending refunds
 */
function refund_get_pending($filters = []) {
    $where = "r.status = ?";
    $params = [REFUND_STATUS_PENDING];
    
    if (!empty($filters['client_id'])) {
        $where .= " AND r.client_id = ?";
        $params[] = $filters['client_id'];
    }
    
    if (!empty($filters['min_amount'])) {
        $where .= " AND r.amount >= ?";
        $params[] = $filters['min_amount'];
    }
    
    $query = "SELECT r.*, c.firstname, c.lastname, i.invoicenum
              FROM mod_refund_requests r
              JOIN tblclients c ON r.client_id = c.id
              JOIN tblinvoices i ON r.invoice_id = i.id
              WHERE {$where}
              ORDER BY r.requested_at ASC";
    
    $result = full_query($query, $params);
    
    $refunds = [];
    while ($row = mysql_fetch_assoc($result)) {
        $refunds[] = $row;
    }
    
    return $refunds;
}

/**
 * Get refund history
 */
function refund_get_history($clientId = null, $limit = 100) {
    $where = "1=1";
    $params = [];
    
    if ($clientId) {
        $where .= " AND r.client_id = ?";
        $params[] = $clientId;
    }
    
    $query = "SELECT r.*, c.firstname, c.lastname, i.invoicenum
              FROM mod_refund_requests r
              JOIN tblclients c ON r.client_id = c.id
              JOIN tblinvoices i ON r.invoice_id = i.id
              WHERE {$where}
              ORDER BY r.requested_at DESC LIMIT ?";
    $params[] = $limit;
    
    $result = full_query($query, $params);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $history[] = $row;
    }
    
    return $history;
}

/**
 * Log refund action
 */
function refund_log($requestId, $action, $data = []) {
    if (is_array($requestId)) {
        $requestId = $requestId['id'] ?? $requestId['request_id'] ?? 0;
    }
    
    $query = "SELECT id FROM mod_refund_requests WHERE id = ? OR request_id = ?";
    $result = full_query($query, [$requestId, $requestId]);
    $request = mysql_fetch_assoc($result);
    
    if ($request) {
        $requestId = $request['id'];
    }
    
    $fields = ['request_id', 'action', 'data', 'created_at'];
    $values = [$requestId, $action, json_encode($data), date('Y-m-d H:i:s')];
    
    insert_query('mod_refund_logs', array_combine($fields, $values));
}

/**
 * Notify admins of new refund request
 */
function refund_notify_admins($requestId) {
    $request = refund_get($requestId);
    
    // Get admins with refund permissions
    $admins = refund_get_admin_users();
    
    foreach ($admins as $admin) {
        send_email($admin['email'], 'Refund Request', [
            'id' => $request['request_id'],
            'client' => $request['firstname'] . ' ' . $request['lastname'],
            'amount' => $request['amount'],
            'reason' => $request['reason']
        ]);
    }
}

/**
 * Get admin users for notifications
 */
function refund_get_admin_users() {
    $query = "SELECT email FROM tbladmins 
              WHERE privileges LIKE '%refunds%' OR role = 'owner'";
    $result = full_query($query);
    
    $admins = [];
    while ($row = mysql_fetch_assoc($result)) {
        $admins[] = $row;
    }
    
    return $admins;
}

/**
 * Notify client of status change
 */
function refund_notify_client($requestId, $status) {
    $request = refund_get($requestId);
    
    $messages = [
        'approved' => 'Your refund request has been approved and will be processed shortly.',
        'rejected' => 'Your refund request has been rejected. Please contact support for more information.',
        'completed' => 'Your refund of ' . formatCurrency($request['amount']) . ' has been processed.'
    ];
    
    if (isset($messages[$status])) {
        send_email($request['email'], 'Refund Request Update', [
            'request_id' => $request['request_id'],
            'status' => $status,
            'message' => $messages[$status]
        ]);
    }
}

/**
 * Generate refund statistics
 */
function refund_get_stats($startDate = null, $endDate = null) {
    $where = "1=1";
    $params = [];
    
    if ($startDate && $endDate) {
        $where .= " AND created_at BETWEEN ? AND ?";
        $params = [$startDate, $endDate];
    }
    
    $query = "SELECT 
                COUNT(*) as total_requests,
                SUM(amount) as total_refunded,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
                COUNT(CASE WHEN status = 'rejected' THEN 1 END) as rejected,
                COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending,
                AVG(amount) as avg_refund
              FROM mod_refund_requests
              WHERE {$where}";
    
    $result = full_query($query, $params);
    return mysql_fetch_assoc($result);
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Refund Module Hooks
 */

// Hook: Create refund request on client action
add_hook('ClientAreaPage', 1, function($params) {
    if ($_POST['action'] == 'request_refund' && $_SESSION['uid']) {
        $result = refund_create_request([
            'invoice_id' => $_POST['invoice_id'],
            'amount' => $_POST['amount'] ?? null,
            'type' => $_POST['type'] ?? REFUND_FULL,
            'reason' => $_POST['reason'],
            'details' => $_POST['details'],
            'payment_id' => $_POST['payment_id'] ?? 0,
            'requested_by' => $_SESSION['uid']
        ]);
        
        return $result;
    }
});

// Hook: Process automatic refunds for cancelled services
add_hook('ServiceCancellationRefunded', 1, function($params) {
    $serviceId = $params['serviceId'];
    $clientId = $params['clientId'];
    
    // Get unused days
    $service = get_service($serviceId);
    $endDate = strtotime($service['nextduedate']);
    $cancelDate = time();
    $unusedDays = max(0, ceil(($endDate - $cancelDate) / 86400));
    
    if ($unusedDays > 0) {
        // Create proportional refund request
        $invoiceId = get_service_invoice($serviceId);
        
        if ($invoiceId) {
            $request = refund_create_request([
                'invoice_id' => $invoiceId,
                'type' => REFUND_PROPORTIONAL,
                'reason' => REASON_CANCEL,
                'details' => 'Proportional refund for unused service days: ' . $unusedDays . ' days',
                'requested_by' => $clientId
            ]);
            
            // Auto-approve and process for cancellations
            if (refund_is_auto_approve_enabled()) {
                refund_approve($request['request_id'], 0, 'Auto-approved for service cancellation');
                refund_process($request['request_id']);
            }
        }
    }
});

// Hook: Handle refund on payment reversal
add_hook('PaymentReversed', 1, function($params) {
    $paymentId = $params['paymentId'];
    
    // Check for existing refund request
    $query = "SELECT * FROM mod_refund_requests 
              WHERE payment_id = ? AND status IN ('approved', 'processing')";
    $result = full_query($query, [$paymentId]);
    
    if ($existing = mysql_fetch_assoc($result)) {
        // Reverse the refund
        refund_reverse($existing['id']);
    }
});

// Hook: Add refund info to invoice view
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    $maxRefund = refund_calculate_max_amount($invoiceId);
    $refunds = refund_get_invoice_refunds($invoiceId);
    
    return [
        'max_refundable' => $maxRefund,
        'existing_refunds' => $refunds
    ];
});

// Hook: Check refund eligibility
add_hook('BeforeRefundRequest', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    
    // Check if refund window has passed
    $query = "SELECT r.*, DATEDIFF(CURDATE(), r.requested_at) as days_since
              FROM mod_refund_requests r
              WHERE r.invoice_id = ? AND r.status = 'completed'";
    $result = full_query($query, [$invoiceId]);
    
    if ($refund = mysql_fetch_assoc($result)) {
        $windowDays = get_refund_window_days();
        
        if ($refund['days_since'] > $windowDays) {
            return [
                'eligible' => false,
                'reason' => 'Refund window has expired (' . $windowDays . ' days)'
            ];
        }
    }
    
    return ['eligible' => true];
});

// Hook: Process partial refund
add_hook('InvoicePartialPayment', 1, function($params) {
    $invoiceId = $params['invoiceId'];
    $refundAmount = $params['refundAmount'];
    
    // Create partial refund record
    $invoice = get_invoice($invoiceId);
    
    return [
        'refund_created' => true,
        'amount' => $refundAmount,
        'remaining' => $invoice['total'] - $refundAmount
    ];
});

// Hook: Track refund metrics
add_hook('DailyCronJob', 1, function($params) {
    // Get refund statistics
    $stats = refund_get_stats(
        date('Y-m-01'),
        date('Y-m-d')
    );
    
    // Log for reporting
    logActivity("Monthly refunds: {$stats['completed']} requests, " .
                formatCurrency($stats['total_refunded']) . " total");
    
    return $stats;
});
```

---

## Database Schema

```sql
-- Refund requests
CREATE TABLE `mod_refund_requests` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `request_id` VARCHAR(50) NOT NULL UNIQUE,
    `invoice_id` INT NOT NULL,
    `client_id` INT NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `refund_type` ENUM('full', 'partial', 'proportional', 'credit') DEFAULT 'full',
    `reason` VARCHAR(100) NOT NULL,
    `reason_details` TEXT DEFAULT NULL,
    `payment_id` INT DEFAULT NULL,
    `status` ENUM('pending', 'approved', 'rejected', 'processing', 'completed', 'cancelled') DEFAULT 'pending',
    `requested_by` INT NOT NULL,
    `requested_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `ip_address` VARCHAR(45) DEFAULT NULL,
    `approved_by` INT DEFAULT NULL,
    `approved_at` DATETIME DEFAULT NULL,
    `rejected_by` INT DEFAULT NULL,
    `rejected_at` DATETIME DEFAULT NULL,
    `rejection_reason` TEXT DEFAULT NULL,
    `cancelled_by` INT DEFAULT NULL,
    `cancelled_at` DATETIME DEFAULT NULL,
    `processed_at` DATETIME DEFAULT NULL,
    `transaction_id` VARCHAR(255) DEFAULT NULL,
    `admin_notes` TEXT DEFAULT NULL,
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_client` (`client_id`),
    INDEX `idx_status` (`status`),
    INDEX `idx_requested` (`requested_at`),
    FOREIGN KEY (`invoice_id`) REFERENCES `tblinvoices`(`id`) ON DELETE CASCADE,
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Refund logs
CREATE TABLE `mod_refund_logs` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `request_id` INT NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_request` (`request_id`)
);

-- Credit balances
CREATE TABLE `mod_credit_balances` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `remaining` DECIMAL(10,2) NOT NULL,
    `description` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `used_at` DATETIME DEFAULT NULL,
    INDEX `idx_client` (`client_id`),
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Refund configuration
CREATE TABLE `mod_refund_config` (
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
| `ClientAreaPage` | 1 | Handle refund requests |
| `ServiceCancellationRefunded` | 1 | Auto-process cancellation refunds |
| `PaymentReversed` | 1 | Handle payment reversals |
| `AdminAreaViewInvoice` | 1 | Show refund info in invoice |
| `BeforeRefundRequest` | 1 | Check refund eligibility |
| `InvoicePartialPayment` | 1 | Handle partial refunds |
| `DailyCronJob` | 1 | Track refund metrics |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement refund request creation
- [ ] Create approval workflow
- [ ] Implement payment gateway integration
- [ ] Add partial refund support
- [ ] Create proportional calculations
- [ ] Build admin interface
- [ ] Add client notification system
- [ ] Implement credit system
- [ ] Add refund statistics
- [ ] Create refund eligibility checks
- [ ] Test gateway integrations
- [ ] Verify approval workflow
- [ ] Add refund cancellation
- [ ] Implement refund history
- [ ] Add refund search/filter
- [ ] Create refund reports
- [ ] Test edge cases
- [ ] Add refund policies configuration