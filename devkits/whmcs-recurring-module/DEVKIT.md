# WHMCS Recurring Module DevKit

## Header

**Purpose:** Advanced recurring billing module that handles complex recurring billing scenarios including variable billing amounts, milestone-based billing, and subscription cycling management.

**Module Type:** Billing/Automation Module

**Use Case:** Hosting companies needing flexible recurring billing options beyond standard WHMCS, such as usage-based billing, tiered recurring, or custom billing schedules.

---

## Complete Code Template

### File Structure
```
whmcs-recurring-module/
├── README.md
├── DEVKIT.md
├── recurring.php          # Main recurring logic
├── hooks.php             # WHMCS hook integrations
├── schedule_manager.php  # Billing schedule management
└── templates/
    └── admin_recurring.tpl
```

### Main Module File: recurring.php

```php
<?php
/**
 * WHMCS Recurring Billing Module
 * 
 * Provides advanced recurring billing management.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('RECURRING_MODULE_VERSION', '1.0.0');

// Recurring Status
define('RECURRING_STATUS_ACTIVE', 'active');
define('RECURRING_STATUS_PAUSED', 'paused');
define('RECURRING_STATUS_CANCELLED', 'cancelled');
define('RECURRING_STATUS_EXPIRED', 'expired');
define('RECURRING_STATUS_PENDING', 'pending');

// Recurring Types
define('RECURRING_FIXED', 'fixed');
define('RECURRING_VARIABLE', 'variable');
define('RECURRING_MILESTONE', 'milestone');
define('RECURRING_USAGE', 'usage');

// Billing Cycles
define('BILLING_DAILY', 'daily');
define('BILLING_WEEKLY', 'weekly');
define('BILLING_BIWEEKLY', 'biweekly');
define('BILLING_MONTHLY', 'monthly');
define('BILLING_QUARTERLY', 'quarterly');
define('BILLING_SEMIANNUAL', 'semiannual');
define('BILLING_ANNUAL', 'annual');
define('BILLING_BIENNIAL', 'biennial');
define('BILLING_CUSTOM', 'custom');

/**
 * Create recurring billing profile
 */
function recurring_create_profile($clientId, $serviceId, $data) {
    $profileId = recurring_generate_profile_id();
    
    $fields = [
        'profile_id', 'client_id', 'service_id', 'type',
        'billing_cycle', 'base_amount', 'current_amount',
        'start_date', 'next_billing_date', 'end_date',
        'occurrence_count', 'max_occurrences', 'status',
        'created_at', 'updated_at'
    ];
    
    $nextDate = recurring_calculate_next_date(
        $data['start_date'] ?? date('Y-m-d'),
        $data['billing_cycle'] ?? BILLING_MONTHLY
    );
    
    $values = [
        $profileId, $clientId, $serviceId, $data['type'] ?? RECURRING_FIXED,
        $data['billing_cycle'] ?? BILLING_MONTHLY,
        $data['base_amount'], $data['current_amount'] ?? $data['base_amount'],
        $data['start_date'] ?? date('Y-m-d'),
        $nextDate,
        $data['end_date'] ?? null,
        0, $data['max_occurrences'] ?? 0,
        RECURRING_STATUS_ACTIVE, date('Y-m-d H:i:s'), date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_recurring_profiles', array_combine($fields, $values));
    
    $profileDbId = mysql_insert_id();
    
    // Create first invoice
    recurring_create_invoice($profileDbId);
    
    recurring_log_event($profileDbId, 'profile_created', [
        'client_id' => $clientId,
        'service_id' => $serviceId,
        'amount' => $data['base_amount']
    ]);
    
    return [
        'success' => true,
        'profile_id' => $profileId,
        'db_id' => $profileDbId,
        'next_billing_date' => $nextDate
    ];
}

/**
 * Generate unique profile ID
 */
function recurring_generate_profile_id() {
    return 'RBP-' . strtoupper(substr(md5(uniqid()), 0, 8)) . '-' . date('ymd');
}

/**
 * Calculate next billing date
 */
function recurring_calculate_next_date($fromDate, $cycle) {
    $fromTs = strtotime($fromDate);
    
    switch ($cycle) {
        case BILLING_DAILY:
            return date('Y-m-d', strtotime('+1 day', $fromTs));
        case BILLING_WEEKLY:
            return date('Y-m-d', strtotime('+1 week', $fromTs));
        case BILLING_BIWEEKLY:
            return date('Y-m-d', strtotime('+2 weeks', $fromTs));
        case BILLING_MONTHLY:
            return date('Y-m-d', strtotime('+1 month', $fromTs));
        case BILLING_QUARTERLY:
            return date('Y-m-d', strtotime('+3 months', $fromTs));
        case BILLING_SEMIANNUAL:
            return date('Y-m-d', strtotime('+6 months', $fromTs));
        case BILLING_ANNUAL:
            return date('Y-m-d', strtotime('+1 year', $fromTs));
        case BILLING_BIENNIAL:
            return date('Y-m-d', strtotime('+2 years', $fromTs));
        case BILLING_CUSTOM:
            // Custom interval in days
            return date('Y-m-d', $fromTs + (recurring_get_custom_days($cycle) * 86400));
        default:
            return date('Y-m-d', strtotime('+1 month', $fromTs));
    }
}

/**
 * Get custom interval days
 */
function recurring_get_custom_days($cycle) {
    // Extract days from custom cycle (format: 'custom_X')
    if (preg_match('/custom_(\d+)/', $cycle, $matches)) {
        return (int)$matches[1];
    }
    return 30;
}

/**
 * Create invoice for recurring profile
 */
function recurring_create_invoice($profileId, $amountOverride = null) {
    $profile = recurring_get_profile($profileId);
    
    if (!$profile || $profile['status'] != RECURRING_STATUS_ACTIVE) {
        return ['success' => false, 'error' => 'Profile not active'];
    }
    
    // Calculate amount
    $amount = $amountOverride ?? recurring_calculate_amount($profile);
    
    $invoiceData = [
        'userid' => $profile['client_id'],
        'date' => date('Y-m-d'),
        'duedate' => $profile['next_billing_date'],
        'status' => 'Unpaid',
        'paymentmethod' => 'recurring',
        'notes' => 'Recurring billing: ' . $profile['profile_id']
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    // Add line item
    $service = recurring_get_service($profile['service_id']);
    
    insert_query('tblinvoiceitems', [
        'invoiceid' => $invoiceId,
        'userid' => $profile['client_id'],
        'description' => 'Recurring: ' . ($service['name'] ?? 'Service') . ' (' . recurring_get_cycle_label($profile['billing_cycle']) . ')',
        'amount' => $amount,
        'taxed' => 0
    ]);
    
    // Link invoice to profile
    insert_query('mod_recurring_invoices', [
        'profile_id' => $profileId,
        'invoice_id' => $invoiceId,
        'amount' => $amount,
        'billing_date' => $profile['next_billing_date']
    ]);
    
    return [
        'success' => true,
        'invoice_id' => $invoiceId,
        'amount' => $amount
    ];
}

/**
 * Calculate billing amount
 */
function recurring_calculate_amount($profile) {
    $amount = $profile['current_amount'];
    
    switch ($profile['type']) {
        case RECURRING_VARIABLE:
            // Apply any variable adjustments
            $adjustments = recurring_get_adjustments($profile['id']);
            foreach ($adjustments as $adj) {
                if ($adj['type'] == 'percentage') {
                    $amount *= (1 + ($adj['value'] / 100));
                } else {
                    $amount += $adj['value'];
                }
            }
            break;
            
        case RECURRING_MILESTONE:
            // Get current milestone
            $milestone = recurring_get_current_milestone($profile['id']);
            if ($milestone) {
                $amount = $milestone['amount'];
            }
            break;
            
        case RECURRING_USAGE:
            // Calculate usage-based amount
            $amount = recurring_calculate_usage($profile);
            break;
    }
    
    return round($amount, 2);
}

/**
 * Get profile by ID
 */
function recurring_get_profile($profileId) {
    $query = "SELECT r.*, c.firstname, c.lastname, c.email
              FROM mod_recurring_profiles r
              JOIN tblclients c ON r.client_id = c.id
              WHERE r.id = ? OR r.profile_id = ?";
    $result = full_query($query, [$profileId, $profileId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get service details
 */
function recurring_get_service($serviceId) {
    $query = "SELECT h.*, p.name FROM tblhosting h
              JOIN tblproducts p ON h.packageid = p.id
              WHERE h.id = ?";
    $result = full_query($query, [$serviceId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get billing cycle label
 */
function recurring_get_cycle_label($cycle) {
    $labels = [
        'daily' => 'Daily',
        'weekly' => 'Weekly',
        'biweekly' => 'Bi-Weekly',
        'monthly' => 'Monthly',
        'quarterly' => 'Quarterly',
        'semiannual' => 'Semi-Annual',
        'annual' => 'Annual',
        'biennial' => 'Biennial'
    ];
    
    if (preg_match('/custom_(\d+)/', $cycle, $matches)) {
        return 'Every ' . $matches[1] . ' days';
    }
    
    return $labels[$cycle] ?? 'Monthly';
}

/**
 * Get adjustments for profile
 */
function recurring_get_adjustments($profileId) {
    $query = "SELECT * FROM mod_recurring_adjustments 
              WHERE profile_id = ? AND effective_date <= CURDATE()
              ORDER BY effective_date DESC";
    $result = full_query($query, [$profileId]);
    
    $adjustments = [];
    while ($row = mysql_fetch_assoc($result)) {
        $adjustments[] = $row;
    }
    
    return $adjustments;
}

/**
 * Get current milestone
 */
function recurring_get_current_milestone($profileId) {
    $query = "SELECT * FROM mod_recurring_milestones
              WHERE profile_id = ? AND status = 'active'
              ORDER BY sequence ASC LIMIT 1";
    $result = full_query($query, [$profileId]);
    return mysql_fetch_assoc($result);
}

/**
 * Calculate usage-based amount
 */
function recurring_calculate_usage($profile) {
    $startDate = date('Y-m-01');
    $endDate = date('Y-m-t');
    
    $query = "SELECT SUM(metric_value) as total_usage, metric_type
              FROM mod_usage_records
              WHERE client_id = ? AND service_id = ?
              AND recorded_at BETWEEN ? AND ?
              GROUP BY metric_type";
    $result = full_query($query, [
        $profile['client_id'], $profile['service_id'],
        $startDate . ' 00:00:00', $endDate . ' 23:59:59'
    ]);
    
    $totalAmount = 0;
    
    while ($usage = mysql_fetch_assoc($result)) {
        $rate = recurring_get_usage_rate($profile['service_id'], $usage['metric_type']);
        $totalAmount += $usage['total_usage'] * $rate;
    }
    
    // Add base amount
    return $totalAmount + $profile['base_amount'];
}

/**
 * Get usage rate for metric
 */
function recurring_get_usage_rate($serviceId, $metricType) {
    $query = "SELECT rate FROM mod_usage_rates 
              WHERE service_id = ? AND metric_type = ?";
    $result = full_query($query, [$serviceId, $metricType]);
    $rate = mysql_fetch_assoc($result);
    
    return $rate['rate'] ?? 0;
}

/**
 * Process recurring billing
 */
function recurring_process($profileId) {
    $profile = recurring_get_profile($profileId);
    
    if (!$profile || $profile['status'] != RECURRING_STATUS_ACTIVE) {
        return ['success' => false, 'error' => 'Profile not eligible'];
    }
    
    // Check if this is the next billing date
    if ($profile['next_billing_date'] > date('Y-m-d')) {
        return ['success' => false, 'error' => 'Not due for billing'];
    }
    
    // Create invoice
    $result = recurring_create_invoice($profileId);
    
    if ($result['success']) {
        // Update occurrence count
        update_query('mod_recurring_profiles', [
            'occurrence_count' => $profile['occurrence_count'] + 1,
            'next_billing_date' => recurring_calculate_next_date(
                $profile['next_billing_date'],
                $profile['billing_cycle']
            ),
            'updated_at' => date('Y-m-d H:i:s')
        ], ['id' => $profileId]);
        
        // Check for max occurrences
        $updatedProfile = recurring_get_profile($profileId);
        if ($updatedProfile['max_occurrences'] > 0 && 
            $updatedProfile['occurrence_count'] >= $updatedProfile['max_occurrences']) {
            recurring_complete_profile($profileId);
        }
        
        recurring_log_event($profileId, 'invoice_created', [
            'invoice_id' => $result['invoice_id'],
            'amount' => $result['amount']
        ]);
    }
    
    return $result;
}

/**
 * Complete recurring profile
 */
function recurring_complete_profile($profileId) {
    update_query('mod_recurring_profiles', [
        'status' => RECURRING_STATUS_EXPIRED,
        'end_date' => date('Y-m-d'),
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $profileId]);
    
    recurring_log_event($profileId, 'profile_completed', [
        'reason' => 'Max occurrences reached'
    ]);
}

/**
 * Pause recurring profile
 */
function recurring_pause($profileId, $resumeDate = null) {
    $profile = recurring_get_profile($profileId);
    
    if (!$profile) {
        return ['success' => false, 'error' => 'Profile not found'];
    }
    
    update_query('mod_recurring_profiles', [
        'status' => RECURRING_STATUS_PAUSED,
        'pause_date' => date('Y-m-d'),
        'resume_date' => $resumeDate,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $profileId]);
    
    recurring_log_event($profileId, 'profile_paused', [
        'resume_date' => $resumeDate
    ]);
    
    return ['success' => true];
}

/**
 * Resume paused profile
 */
function recurring_resume($profileId) {
    $profile = recurring_get_profile($profileId);
    
    if (!$profile || $profile['status'] != RECURRING_STATUS_PAUSED) {
        return ['success' => false, 'error' => 'Profile not paused'];
    }
    
    // Calculate next billing date from today
    $nextDate = recurring_calculate_next_date(date('Y-m-d'), $profile['billing_cycle']);
    
    update_query('mod_recurring_profiles', [
        'status' => RECURRING_STATUS_ACTIVE,
        'pause_date' => null,
        'resume_date' => null,
        'next_billing_date' => $nextDate,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $profileId]);
    
    recurring_log_event($profileId, 'profile_resumed', [
        'new_next_date' => $nextDate
    ]);
    
    return ['success' => true, 'next_billing_date' => $nextDate];
}

/**
 * Cancel recurring profile
 */
function recurring_cancel($profileId, $reason = '') {
    update_query('mod_recurring_profiles', [
        'status' => RECURRING_STATUS_CANCELLED,
        'end_date' => date('Y-m-d'),
        'cancel_reason' => $reason,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $profileId]);
    
    recurring_log_event($profileId, 'profile_cancelled', [
        'reason' => $reason
    ]);
    
    return ['success' => true];
}

/**
 * Update recurring amount
 */
function recurring_update_amount($profileId, $newAmount, $effectiveDate = null) {
    $effectiveDate = $effectiveDate ?? date('Y-m-d');
    
    insert_query('mod_recurring_adjustments', [
        'profile_id' => $profileId,
        'type' => 'fixed',
        'value' => $newAmount - recurring_get_profile($profileId)['current_amount'],
        'effective_date' => $effectiveDate,
        'created_at' => date('Y-m-d H:i:s')
    ]);
    
    update_query('mod_recurring_profiles', [
        'current_amount' => $newAmount,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $profileId]);
    
    recurring_log_event($profileId, 'amount_updated', [
        'new_amount' => $newAmount,
        'effective_date' => $effectiveDate
    ]);
    
    return ['success' => true];
}

/**
 * Add milestone to profile
 */
function recurring_add_milestone($profileId, $data) {
    $data['profile_id'] = $profileId;
    $data['status'] = 'active';
    
    $fields = ['profile_id', 'name', 'sequence', 'amount', 'due_date', 'status', 'created_at'];
    $values = [
        $profileId, $data['name'], $data['sequence'], $data['amount'],
        $data['due_date'], 'active', date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_recurring_milestones', array_combine($fields, $values));
    
    return ['success' => true];
}

/**
 * Process milestone completion
 */
function recurring_complete_milestone($milestoneId, $invoiceId = null) {
    update_query('mod_recurring_milestones', [
        'status' => 'completed',
        'invoice_id' => $invoiceId,
        'completed_at' => date('Y-m-d H:i:s')
    ], ['id' => $milestoneId]);
    
    // Activate next milestone
    $query = "SELECT * FROM mod_recurring_milestones 
              WHERE profile_id = (SELECT profile_id FROM mod_recurring_milestones WHERE id = ?)
              AND status = 'active' ORDER BY sequence ASC LIMIT 1";
    $result = full_query($query, [$milestoneId]);
    
    if ($next = mysql_fetch_assoc($result)) {
        update_query('mod_recurring_profiles', [
            'current_amount' => $next['amount']
        ], ['id' => $next['profile_id']]);
    }
    
    return ['success' => true];
}

/**
 * Log recurring event
 */
function recurring_log_event($profileId, $event, $data = []) {
    $fields = ['profile_id', 'event', 'data', 'created_at'];
    $values = [$profileId, $event, json_encode($data), date('Y-m-d H:i:s')];
    
    insert_query('mod_recurring_events', array_combine($fields, $values));
}

/**
 * Get client recurring profiles
 */
function recurring_get_client_profiles($clientId) {
    $query = "SELECT * FROM mod_recurring_profiles 
              WHERE client_id = ? ORDER BY created_at DESC";
    $result = full_query($query, [$clientId]);
    
    $profiles = [];
    while ($row = mysql_fetch_assoc($result)) {
        $profiles[] = $row;
    }
    
    return $profiles;
}

/**
 * Process all due recurring profiles (cron)
 */
function recurring_process_all_due() {
    $query = "SELECT id FROM mod_recurring_profiles 
              WHERE status = 'active' AND next_billing_date <= CURDATE()";
    $result = full_query($query);
    
    $processed = 0;
    $failed = 0;
    
    while ($profile = mysql_fetch_assoc($result)) {
        $result = recurring_process($profile['id']);
        
        if ($result['success']) {
            $processed++;
        } else {
            $failed++;
        }
    }
    
    return [
        'processed' => $processed,
        'failed' => $failed
    ];
}

/**
 * Resume paused profiles that are due
 */
function recurring_process_paused() {
    $query = "SELECT id FROM mod_recurring_profiles 
              WHERE status = 'paused' AND resume_date <= CURDATE()";
    $result = full_query($query);
    
    $resumed = 0;
    
    while ($profile = mysql_fetch_assoc($result)) {
        recurring_resume($profile['id']);
        $resumed++;
    }
    
    return ['resumed' => $resumed];
}

/**
 * Get recurring statistics
 */
function recurring_get_stats($startDate = null, $endDate = null) {
    $where = "1=1";
    $params = [];
    
    if ($startDate && $endDate) {
        $where .= " AND created_at BETWEEN ? AND ?";
        $params = [$startDate, $endDate];
    }
    
    $query = "SELECT 
                COUNT(*) as total_profiles,
                SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN status = 'paused' THEN 1 ELSE 0 END) as paused,
                SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
                SUM(CASE WHEN status = 'expired' THEN 1 ELSE 0 END) as expired,
                SUM(current_amount) as total_monthly_value,
                SUM(CASE WHEN status = 'active' THEN current_amount ELSE 0 END) as active_mrr
              FROM mod_recurring_profiles
              WHERE {$where}";
    
    $result = full_query($query, $params);
    return mysql_fetch_assoc($result);
}

/**
 * Get recurring profile history
 */
function recurring_get_history($profileId) {
    $query = "SELECT * FROM mod_recurring_events 
              WHERE profile_id = ? ORDER BY created_at DESC";
    $result = full_query($query, [$profileId]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['data'] = json_decode($row['data'], true);
        $history[] = $row;
    }
    
    return $history;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Recurring Module Hooks
 */

// Hook: Create recurring profile on service creation
add_hook('ServiceCreated', 1, function($params) {
    $serviceId = $params['serviceId'];
    $clientId = $params['userId'];
    $productId = $params['productId'];
    
    // Check if product should have recurring profile
    if (!recurring_product_enabled($productId)) {
        return [];
    }
    
    $service = recurring_get_service($serviceId);
    
    $result = recurring_create_profile($clientId, $serviceId, [
        'type' => RECURRING_FIXED,
        'billing_cycle' => $service['billingcycle'] ?? BILLING_MONTHLY,
        'base_amount' => $service['amount'],
        'start_date' => $service['nextduedate']
    ]);
    
    return $result;
});

// Hook: Process recurring profiles daily
add_hook('DailyCronJob', 1, function($params) {
    $result = recurring_process_all_due();
    
    logActivity("Recurring billing processed: {$result['processed']} profiles, {$result['failed']} failed");
    
    return $result;
});

// Hook: Resume paused profiles
add_hook('DailyCronJob', 2, function($params) {
    $result = recurring_process_paused();
    
    return $result;
});

// Hook: Handle payment success
add_hook('InvoicePaid', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check for recurring invoice link
    $query = "SELECT profile_id FROM mod_recurring_invoices WHERE invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    if ($link = mysql_fetch_assoc($result)) {
        // Check for pending milestones
        recurring_check_milestones($link['profile_id'], $invoiceId);
    }
});

// Hook: Cancel recurring on service termination
add_hook('ServiceTermination', 1, function($params) {
    $serviceId = $params['serviceId'];
    
    $query = "SELECT id FROM mod_recurring_profiles WHERE service_id = ?";
    $result = full_query($query, [$serviceId]);
    
    while ($profile = mysql_fetch_assoc($result)) {
        recurring_cancel($profile['id'], 'Service terminated');
    }
});

// Hook: Update recurring on service upgrade
add_hook('ServiceChangePackage', 1, function($params) {
    $serviceId = $params['serviceId'];
    $newAmount = $params['newAmount'];
    
    $query = "SELECT id FROM mod_recurring_profiles WHERE service_id = ?";
    $result = full_query($query, [$serviceId]);
    
    if ($profile = mysql_fetch_assoc($result)) {
        recurring_update_amount($profile['id'], $newAmount);
    }
});

// Hook: Pause recurring on client request
add_hook('ClientAreaPage', 1, function($params) {
    if ($_POST['action'] == 'pause_recurring' && $_SESSION['uid']) {
        $profileId = $_POST['profile_id'];
        $resumeDate = $_POST['resume_date'] ?? null;
        
        return recurring_pause($profileId, $resumeDate);
    }
    
    if ($_POST['action'] == 'resume_recurring' && $_SESSION['uid']) {
        $profileId = $_POST['profile_id'];
        
        return recurring_resume($profileId);
    }
});

// Hook: Add recurring info to invoice
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    $query = "SELECT rp.* FROM mod_recurring_invoices ri
              JOIN mod_recurring_profiles rp ON ri.profile_id = rp.id
              WHERE ri.invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    if ($profile = mysql_fetch_assoc($result)) {
        return [
            'recurring_profile' => $profile,
            'billing_cycle' => recurring_get_cycle_label($profile['billing_cycle'])
        ];
    }
});
```

---

## Database Schema

```sql
-- Recurring profiles
CREATE TABLE `mod_recurring_profiles` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `profile_id` VARCHAR(50) NOT NULL UNIQUE,
    `client_id` INT NOT NULL,
    `service_id` INT NOT NULL,
    `type` ENUM('fixed', 'variable', 'milestone', 'usage') DEFAULT 'fixed',
    `billing_cycle` VARCHAR(20) DEFAULT 'monthly',
    `base_amount` DECIMAL(10,2) NOT NULL,
    `current_amount` DECIMAL(10,2) NOT NULL,
    `start_date` DATE NOT NULL,
    `next_billing_date` DATE NOT NULL,
    `end_date` DATE DEFAULT NULL,
    `pause_date` DATE DEFAULT NULL,
    `resume_date` DATE DEFAULT NULL,
    `occurrence_count` INT DEFAULT 0,
    `max_occurrences` INT DEFAULT 0,
    `status` ENUM('active', 'paused', 'cancelled', 'expired', 'pending') DEFAULT 'active',
    `cancel_reason` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_client` (`client_id`),
    INDEX `idx_service` (`service_id`),
    INDEX `idx_status` (`status`),
    INDEX `idx_next_billing` (`next_billing_date`),
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Recurring invoices link
CREATE TABLE `mod_recurring_invoices` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `profile_id` INT NOT NULL,
    `invoice_id` INT NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `billing_date` DATE DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_invoice` (`invoice_id`),
    FOREIGN KEY (`profile_id`) REFERENCES `mod_recurring_profiles`(`id`) ON DELETE CASCADE
);

-- Recurring milestones
CREATE TABLE `mod_recurring_milestones` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `profile_id` INT NOT NULL,
    `name` VARCHAR(255) NOT NULL,
    `sequence` INT NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `due_date` DATE DEFAULT NULL,
    `status` ENUM('active', 'completed', 'skipped') DEFAULT 'active',
    `invoice_id` INT DEFAULT NULL,
    `completed_at` DATETIME DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_profile` (`profile_id`),
    INDEX `idx_status` (`status`),
    FOREIGN KEY (`profile_id`) REFERENCES `mod_recurring_profiles`(`id`) ON DELETE CASCADE
);

-- Recurring adjustments
CREATE TABLE `mod_recurring_adjustments` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `profile_id` INT NOT NULL,
    `type` ENUM('fixed', 'percentage', 'amount') NOT NULL,
    `value` DECIMAL(10,4) NOT NULL,
    `effective_date` DATE NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_profile` (`profile_id`),
    FOREIGN KEY (`profile_id`) REFERENCES `mod_recurring_profiles`(`id`) ON DELETE CASCADE
);

-- Recurring events log
CREATE TABLE `mod_recurring_events` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `profile_id` INT NOT NULL,
    `event` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_profile` (`profile_id`),
    FOREIGN KEY (`profile_id`) REFERENCES `mod_recurring_profiles`(`id`) ON DELETE CASCADE
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ServiceCreated` | 1 | Create recurring profile |
| `DailyCronJob` | 1 | Process due recurring |
| `DailyCronJob` | 2 | Resume paused profiles |
| `InvoicePaid` | 1 | Handle milestone completion |
| `ServiceTermination` | 1 | Cancel recurring |
| `ServiceChangePackage` | 1 | Update recurring amount |
| `ClientAreaPage` | 1 | Handle pause/resume requests |
| `AdminAreaViewInvoice` | 1 | Show recurring info |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement profile creation
- [ ] Create billing cycle calculations
- [ ] Implement invoice generation
- [ ] Add milestone support
- [ ] Create variable billing
- [ ] Add usage-based billing
- [ ] Build pause/resume functionality
- [ ] Create cancellation handling
- [ ] Implement adjustments
- [ ] Build history logging
- [ ] Create statistics reporting
- [ ] Add admin interface
- [ ] Implement client portal views
- [ ] Test billing cycle calculations
- [ ] Verify invoice generation
- [ ] Test milestone progression
- [ ] Add email notifications
- [ ] Create export functionality