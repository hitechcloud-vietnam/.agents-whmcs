# WHMCS Prorate Module DevKit

## Header

**Purpose:** Prorated billing module that handles proportional billing calculations for mid-cycle upgrades, downgrades, cancellations, and billing cycle changes.

**Module Type:** Billing/Calculation Module

**Use Case:** Hosting companies needing accurate prorated billing when customers upgrade/downgrade mid-cycle, change billing frequencies, or cancel services before the billing cycle ends.

---

## Complete Code Template

### File Structure
```
whmcs-prorate-module/
├── README.md
├── DEVKIT.md
├── prorate.php           # Main prorate logic
├── hooks.php            # WHMCS hook integrations
└── templates/
    └── admin_prorate.tpl
```

### Main Module File: prorate.php

```php
<?php
/**
 * WHMCS Prorate Module
 * 
 * Handles prorated billing calculations.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('PRORATE_MODULE_VERSION', '1.0.0');

// Prorate Types
define('PRORATE_UPGRADE', 'upgrade');
define('PRORATE_DOWNGRADE', 'downgrade');
define('PRORATE_CANCEL', 'cancellation');
define('PRORATE_CYCLE_CHANGE', 'cycle_change');

// Billing Cycles
define('BILLING_MONTHLY', 'monthly');
define('BILLING_QUARTERLY', 'quarterly');
define('BILLING_SEMIANNUAL', 'semiannual');
define('BILLING_ANNUAL', 'annual');
define('BILLING_BIENNIAL', 'biennial');

/**
 * Calculate prorated amount for upgrade
 */
function prorate_calculate_upgrade($serviceId, $newProductId, $effectiveDate = null) {
    $effectiveDate = $effectiveDate ?? date('Y-m-d');
    
    $service = prorate_get_service($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    // Get old and new product pricing
    $oldProduct = prorate_get_product($service['packageid']);
    $newProduct = prorate_get_product($newProductId);
    
    // Calculate days remaining in current cycle
    $nextDueDate = strtotime($service['nextduedate']);
    $effectiveTs = strtotime($effectiveDate);
    $daysRemaining = max(0, ceil(($nextDueDate - $effectiveTs) / 86400));
    
    // Get billing cycle info
    $cycleDays = prorate_get_cycle_days($service['billingcycle']);
    $daysUsed = $cycleDays - $daysRemaining;
    
    // Calculate daily rates
    $oldDailyRate = $service['amount'] / $cycleDays;
    
    // Get new pricing for the cycle
    $newPricing = prorate_get_product_pricing($newProductId, $service['billingcycle']);
    $newCyclePrice = $newPricing['price'] ?? 0;
    $newDailyRate = $newCyclePrice / $cycleDays;
    
    // Calculate credit for unused old plan
    $creditAmount = $oldDailyRate * $daysRemaining;
    
    // Calculate cost for new plan
    $chargeAmount = $newDailyRate * $daysRemaining;
    
    // Net prorate amount
    $netAmount = $chargeAmount - $creditAmount;
    
    // Handle setup fees
    $setupFee = $newPricing['setupfee'] ?? 0;
    
    return [
        'success' => true,
        'type' => PRORATE_UPGRADE,
        'service_id' => $serviceId,
        'old_product' => $oldProduct['name'],
        'new_product' => $newProduct['name'],
        'days_remaining' => $daysRemaining,
        'days_used' => $daysUsed,
        'cycle_days' => $cycleDays,
        'credit_amount' => round($creditAmount, 2),
        'charge_amount' => round($chargeAmount, 2),
        'net_amount' => round($netAmount, 2),
        'setup_fee' => round($setupFee, 2),
        'total_charge' => round($netAmount + $setupFee, 2)
    ];
}

/**
 * Get service details
 */
function prorate_get_service($serviceId) {
    $query = "SELECT h.*, c.firstname, c.lastname, p.name as product_name
              FROM tblhosting h
              JOIN tblclients c ON h.userid = c.id
              JOIN tblproducts p ON h.packageid = p.id
              WHERE h.id = ?";
    $result = full_query($query, [$serviceId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get product details
 */
function prorate_get_product($productId) {
    $query = "SELECT * FROM tblproducts WHERE id = ?";
    $result = full_query($query, [$productId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get product pricing for billing cycle
 */
function prorate_get_product_pricing($productId, $cycle) {
    $cycleField = 'monthly';
    switch ($cycle) {
        case BILLING_MONTHLY:
            $cycleField = 'monthly';
            break;
        case BILLING_QUARTERLY:
            $cycleField = 'quarterly';
            break;
        case BILLING_SEMIANNUAL:
            $cycleField = 'semiannual';
            break;
        case BILLING_ANNUAL:
            $cycleField = 'annual';
            break;
        case BILLING_BIENNIAL:
            $cycleField = 'biennial';
            break;
    }
    
    $query = "SELECT * FROM mod_product_pricing WHERE product_id = ?";
    $result = full_query($query, [$productId]);
    $pricing = mysql_fetch_assoc($result);
    
    if (!$pricing) {
        // Fall back to default WHMCS pricing
        $query = "SELECT * FROM tblpricing WHERE relid = ? AND type = 'product'";
        $result = full_query($query, [$productId]);
        $pricing = mysql_fetch_assoc($result);
    }
    
    return [
        'price' => $pricing[$cycleField] ?? $pricing['monthly'] ?? 0,
        'setupfee' => $pricing['setupfee'] ?? 0
    ];
}

/**
 * Get days in billing cycle
 */
function prorate_get_cycle_days($cycle) {
    switch ($cycle) {
        case BILLING_MONTHLY:
            return 30;
        case BILLING_QUARTERLY:
            return 90;
        case BILLING_SEMIANNUAL:
            return 180;
        case BILLING_ANNUAL:
            return 365;
        case BILLING_BIENNIAL:
            return 730;
        default:
            return 30;
    }
}

/**
 * Calculate prorated amount for downgrade
 */
function prorate_calculate_downgrade($serviceId, $newProductId, $effectiveDate = null) {
    $effectiveDate = $effectiveDate ?? date('Y-m-d');
    
    $service = prorate_get_service($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    $oldProduct = prorate_get_product($service['packageid']);
    $newProduct = prorate_get_product($newProductId);
    
    $nextDueDate = strtotime($service['nextduedate']);
    $effectiveTs = strtotime($effectiveDate);
    $daysRemaining = max(0, ceil(($nextDueDate - $effectiveTs) / 86400));
    
    $cycleDays = prorate_get_cycle_days($service['billingcycle']);
    
    // Calculate credit
    $oldDailyRate = $service['amount'] / $cycleDays;
    $creditAmount = $oldDailyRate * $daysRemaining;
    
    // Calculate charge for new plan
    $newPricing = prorate_get_product_pricing($newProductId, $service['billingcycle']);
    $newDailyRate = ($newPricing['price'] ?? 0) / $cycleDays;
    $chargeAmount = $newDailyRate * $daysRemaining;
    
    $netAmount = $chargeAmount - $creditAmount;
    
    return [
        'success' => true,
        'type' => PRORATE_DOWNGRADE,
        'service_id' => $serviceId,
        'old_product' => $oldProduct['name'],
        'new_product' => $newProduct['name'],
        'days_remaining' => $daysRemaining,
        'cycle_days' => $cycleDays,
        'credit_amount' => round($creditAmount, 2),
        'charge_amount' => round($chargeAmount, 2),
        'net_amount' => round($netAmount, 2)
    ];
}

/**
 * Calculate prorated refund for cancellation
 */
function prorate_calculate_cancellation($serviceId, $effectiveDate = null, $cancelType = 'immediate') {
    $effectiveDate = $effectiveDate ?? date('Y-m-d');
    
    $service = prorate_get_service($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    $nextDueDate = strtotime($service['nextduedate']);
    $effectiveTs = strtotime($effectiveDate);
    
    if ($cancelType === 'end_of_cycle') {
        // No refund needed, service ends at next due date
        return [
            'success' => true,
            'type' => PRORATE_CANCEL,
            'cancel_type' => 'end_of_cycle',
            'service_id' => $serviceId,
            'refund_amount' => 0,
            'end_date' => $service['nextduedate']
        ];
    }
    
    // Immediate cancellation - calculate refund
    $daysRemaining = max(0, ceil(($nextDueDate - $effectiveTs) / 86400));
    $cycleDays = prorate_get_cycle_days($service['billingcycle']);
    
    // Calculate daily rate
    $dailyRate = $service['amount'] / $cycleDays;
    $refundAmount = $dailyRate * $daysRemaining;
    
    return [
        'success' => true,
        'type' => PRORATE_CANCEL,
        'cancel_type' => 'immediate',
        'service_id' => $serviceId,
        'product_name' => $service['product_name'],
        'amount_paid' => $service['amount'],
        'days_remaining' => $daysRemaining,
        'cycle_days' => $cycleDays,
        'daily_rate' => round($dailyRate, 4),
        'refund_amount' => round($refundAmount, 2)
    ];
}

/**
 * Calculate billing cycle change
 */
function prorate_calculate_cycle_change($serviceId, $newCycle, $effectiveDate = null) {
    $effectiveDate = $effectiveDate ?? date('Y-m-d');
    
    $service = prorate_get_service($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    $oldCycleDays = prorate_get_cycle_days($service['billingcycle']);
    $newCycleDays = prorate_get_cycle_days($newCycle);
    
    // Calculate time used in current cycle
    $regDate = strtotime($service['regdate']);
    $nextDueDate = strtotime($service['nextduedate']);
    $effectiveTs = strtotime($effectiveDate);
    
    // If we're in the middle of a cycle
    $daysUsed = max(0, ceil(($effectiveTs - $regDate) / 86400));
    $daysRemaining = $oldCycleDays - $daysUsed;
    
    // Credit for unused time
    $oldDailyRate = $service['amount'] / $oldCycleDays;
    $creditAmount = $oldDailyRate * $daysRemaining;
    
    // Calculate cost for new cycle
    $newPricing = prorate_get_product_pricing($service['packageid'], $newCycle);
    $newCyclePrice = $newPricing['price'] ?? 0;
    $newMonthlyRate = $newCyclePrice / $newCycleDays;
    
    // Cost for remaining days in new cycle
    $chargeAmount = $newMonthlyRate * ($newCycleDays - $daysRemaining);
    
    $netAmount = $newCyclePrice - $creditAmount;
    
    // Calculate next due date
    $nextDue = prorate_calculate_next_due($newCycle, $effectiveDate);
    
    return [
        'success' => true,
        'type' => PRORATE_CYCLE_CHANGE,
        'service_id' => $serviceId,
        'old_cycle' => $service['billingcycle'],
        'new_cycle' => $newCycle,
        'days_used' => $daysUsed,
        'days_remaining' => $daysRemaining,
        'old_cycle_days' => $oldCycleDays,
        'new_cycle_days' => $newCycleDays,
        'credit_amount' => round($creditAmount, 2),
        'new_cycle_price' => round($newCyclePrice, 2),
        'net_amount' => round($netAmount, 2),
        'next_due_date' => $nextDue
    ];
}

/**
 * Calculate next due date for cycle
 */
function prorate_calculate_next_due($cycle, $fromDate = null) {
    $fromDate = $fromDate ?? date('Y-m-d');
    
    switch ($cycle) {
        case BILLING_MONTHLY:
            return date('Y-m-d', strtotime('+1 month', strtotime($fromDate)));
        case BILLING_QUARTERLY:
            return date('Y-m-d', strtotime('+3 months', strtotime($fromDate)));
        case BILLING_SEMIANNUAL:
            return date('Y-m-d', strtotime('+6 months', strtotime($fromDate)));
        case BILLING_ANNUAL:
            return date('Y-m-d', strtotime('+1 year', strtotime($fromDate)));
        case BILLING_BIENNIAL:
            return date('Y-m-d', strtotime('+2 years', strtotime($fromDate)));
        default:
            return date('Y-m-d', strtotime('+1 month', strtotime($fromDate)));
    }
}

/**
 * Apply prorate changes to service
 */
function prorate_apply_changes($serviceId, $prorateData, $options = []) {
    $service = prorate_get_service($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    // Create prorate invoice if needed
    $invoiceId = null;
    if ($prorateData['net_amount'] != 0 || !empty($prorateData['setup_fee'])) {
        $invoiceId = prorate_create_invoice($service, $prorateData);
    }
    
    // Update service
    $updateData = [
        'nextduedate' => $prorateData['next_due_date'] ?? prorate_calculate_next_due(
            $prorateData['new_cycle'] ?? $service['billingcycle'])
    ];
    
    if (!empty($prorateData['new_product_id'])) {
        $updateData['packageid'] = $prorateData['new_product_id'];
    }
    
    if (!empty($prorateData['new_cycle'])) {
        $updateData['billingcycle'] = $prorateData['new_cycle'];
    }
    
    if (!empty($prorateData['new_amount'])) {
        $updateData['amount'] = $prorateData['new_amount'];
    }
    
    update_query('tblhosting', $updateData, ['id' => $serviceId]);
    
    // Log the change
    prorate_log_change($serviceId, $prorateData, $invoiceId);
    
    return [
        'success' => true,
        'invoice_id' => $invoiceId,
        'service_updated' => true
    ];
}

/**
 * Create prorate invoice
 */
function prorate_create_invoice($service, $prorateData) {
    $amount = $prorateData['net_amount'] ?? 0;
    $setupFee = $prorateData['setup_fee'] ?? 0;
    $total = $amount + $setupFee;
    
    if ($total <= 0) {
        return null;
    }
    
    $invoiceData = [
        'userid' => $service['userid'],
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d'),
        'status' => 'Unpaid',
        'paymentmethod' => $service['paymentmethod'] ?? 'banktransfer',
        'notes' => 'Prorate ' . $prorateData['type'] . ': ' . $prorateData['old_product'] . 
                   ' to ' . ($prorateData['new_product'] ?? $prorateData['product_name'] ?? 'N/A')
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    // Add line items
    $items = [];
    
    if ($prorateData['type'] == PRORATE_UPGRADE) {
        $items[] = [
            'description' => 'Upgrade from ' . $prorateData['old_product'] . ' to ' . $prorateData['new_product'],
            'amount' => $prorateData['net_amount'],
            'taxed' => 0
        ];
    } elseif ($prorateData['type'] == PRORATE_DOWNGRADE) {
        $items[] = [
            'description' => 'Downgrade from ' . $prorateData['old_product'] . ' to ' . $prorateData['new_product'],
            'amount' => $prorateData['net_amount'],
            'taxed' => 0
        ];
    } elseif ($prorateData['type'] == PRORATE_CANCEL) {
        $items[] = [
            'description' => 'Cancellation refund credit',
            'amount' => -$prorateData['refund_amount'],
            'taxed' => 0
        ];
    } elseif ($prorateData['type'] == PRORATE_CYCLE_CHANGE) {
        $items[] = [
            'description' => 'Billing cycle change from ' . $prorateData['old_cycle'] . ' to ' . $prorateData['new_cycle'],
            'amount' => $prorateData['net_amount'],
            'taxed' => 0
        ];
    }
    
    if ($setupFee > 0) {
        $items[] = [
            'description' => 'Setup fee for new plan',
            'amount' => $setupFee,
            'taxed' => 0
        ];
    }
    
    foreach ($items as $item) {
        insert_query('tblinvoiceitems', [
            'invoiceid' => $invoiceId,
            'userid' => $service['userid'],
            'description' => $item['description'],
            'amount' => $item['amount'],
            'taxed' => $item['taxed']
        ]);
    }
    
    return $invoiceId;
}

/**
 * Log prorate change
 */
function prorate_log_change($serviceId, $prorateData, $invoiceId = null) {
    $fields = ['service_id', 'type', 'data', 'invoice_id', 'created_at'];
    $values = [
        $serviceId, $prorateData['type'], json_encode($prorateData),
        $invoiceId, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_prorate_history', array_combine($fields, $values));
}

/**
 * Get prorate history for service
 */
function prorate_get_history($serviceId) {
    $query = "SELECT * FROM mod_prorate_history 
              WHERE service_id = ? ORDER BY created_at DESC";
    $result = full_query($query, [$serviceId]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['data'] = json_decode($row['data'], true);
        $history[] = $row;
    }
    
    return $history;
}

/**
 * Get prorate preview for admin
 */
function prorate_get_preview($serviceId, $newProductId = null, $newCycle = null) {
    $service = prorate_get_service($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    if ($newProductId && $newProductId != $service['packageid']) {
        $oldPrice = $service['amount'];
        $newPricing = prorate_get_product_pricing($newProductId, $service['billingcycle']);
        $newPrice = $newPricing['price'];
        
        if ($newPrice > $oldPrice) {
            return prorate_calculate_upgrade($serviceId, $newProductId);
        } else {
            return prorate_calculate_downgrade($serviceId, $newProductId);
        }
    }
    
    if ($newCycle && $newCycle != $service['billingcycle']) {
        return prorate_calculate_cycle_change($serviceId, $newCycle);
    }
    
    return ['success' => false, 'error' => 'No changes to calculate'];
}

/**
 * Validate prorate calculation
 */
function prorate_validate($prorateData, &$errors = []) {
    if (!$prorateData['success']) {
        $errors[] = $prorateData['error'] ?? 'Calculation failed';
    }
    
    if (isset($prorateData['days_remaining']) && $prorateData['days_remaining'] < 0) {
        $errors[] = 'Invalid days remaining';
    }
    
    if (isset($prorateData['net_amount']) && $prorateData['net_amount'] < -10000) {
        $errors[] = 'Excessive credit amount - verify calculation';
    }
    
    return count($errors) === 0;
}

/**
 * Get billing cycle label
 */
function prorate_get_cycle_label($cycle) {
    $labels = [
        'monthly' => 'Monthly',
        'quarterly' => 'Quarterly',
        'semiannual' => 'Semi-Annual',
        'annual' => 'Annual',
        'biennial' => 'Biennial'
    ];
    
    return $labels[$cycle] ?? 'Monthly';
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Prorate Module Hooks
 */

// Hook: Calculate prorate on product change
add_hook('ServiceChangePackage', 1, function($params) {
    $serviceId = $params['serviceId'];
    $newProductId = $params['newProductId'];
    
    $prorate = prorate_calculate_upgrade($serviceId, $newProductId);
    
    return [
        'prorate_required' => true,
        'amount' => $prorate['total_charge'] ?? $prorate['net_amount'] ?? 0,
        'credit' => $prorate['credit_amount'] ?? 0,
        'charge' => $prorate['charge_amount'] ?? 0
    ];
});

// Hook: Apply prorate on upgrade
add_hook('AfterServiceChangePackage', 1, function($params) {
    $serviceId = $params['serviceId'];
    $newProductId = $params['newProductId'];
    
    $prorate = prorate_get_preview($serviceId, $newProductId);
    
    if ($prorate['success']) {
        // Update service with new pricing
        $newPricing = prorate_get_product_pricing($newProductId, $prorate['old_cycle'] ?? 'monthly');
        
        update_query('tblhosting', [
            'packageid' => $newProductId,
            'amount' => $newPricing['price']
        ], ['id' => $serviceId]);
        
        // Create invoice if needed
        if ($prorate['net_amount'] != 0) {
            $service = prorate_get_service($serviceId);
            prorate_create_invoice($service, $prorate);
        }
    }
});

// Hook: Calculate cancellation refund
add_hook('ServiceTerminationRequest', 1, function($params) {
    $serviceId = $params['serviceId'];
    $cancelType = $params['cancelType'] ?? 'immediate';
    
    $prorate = prorate_calculate_cancellation($serviceId, null, $cancelType);
    
    return [
        'refund_eligible' => $cancelType == 'immediate',
        'refund_amount' => $prorate['refund_amount'] ?? 0,
        'end_date' => $prorate['end_date'] ?? null
    ];
});

// Hook: Apply cancellation prorate
add_hook('ServiceTerminated', 1, function($params) {
    $serviceId = $params['serviceId'];
    
    // Calculate refund
    $prorate = prorate_calculate_cancellation($serviceId);
    
    if ($prorate['refund_amount'] > 0) {
        // Create credit for customer
        $service = prorate_get_service($serviceId);
        
        insert_query('mod_prorate_credits', [
            'service_id' => $serviceId,
            'client_id' => $service['userid'],
            'amount' => $prorate['refund_amount'],
            'description' => 'Cancellation refund for ' . $service['product_name'],
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
});

// Hook: Handle billing cycle change
add_hook('ServiceChangeBillingCycle', 1, function($params) {
    $serviceId = $params['serviceId'];
    $newCycle = $params['newCycle'];
    
    $prorate = prorate_calculate_cycle_change($serviceId, $newCycle);
    
    return [
        'prorate_amount' => $prorate['net_amount'] ?? 0,
        'new_cycle_price' => $prorate['new_cycle_price'] ?? 0,
        'next_due_date' => $prorate['next_due_date'] ?? null
    ];
});

// Hook: Display prorate info in admin
add_hook('AdminAreaViewService', 1, function($params) {
    $serviceId = $params['serviceId'];
    
    $history = prorate_get_history($serviceId);
    $lastChange = !empty($history) ? $history[0] : null;
    
    return [
        'prorate_history' => $history,
        'last_prorate' => $lastChange
    ];
});

// Hook: Validate prorate before change
add_hook('BeforeServiceChange', 1, function($params) {
    $serviceId = $params['serviceId'];
    
    // Check for pending prorate invoices
    $query = "SELECT id, total FROM tblinvoices 
              WHERE userid = ? AND status = 'Unpaid' AND notes LIKE '%Prorate%'";
    $result = full_query($query, [$params['userId']]);
    
    if ($pending = mysql_fetch_assoc($result)) {
        return [
            'blocked' => true,
            'reason' => 'Pending prorate invoice # ' . $pending['id'] . 
                       ' must be paid first (Amount: ' . formatCurrency($pending['total']) . ')'
        ];
    }
});

// Hook: Update renewal amount after change
add_hook('ServiceChangeCompleted', 1, function($params) {
    $serviceId = $params['serviceId'];
    
    $service = prorate_get_service($serviceId);
    
    // Recalculate next renewal amount
    $renewalAmount = prorate_get_renewal_amount($service);
    
    return [
        'new_renewal_amount' => $renewalAmount,
        'next_due_date' => $service['nextduedate']
    ];
});
```

---

## Database Schema

```sql
-- Prorate history
CREATE TABLE `mod_prorate_history` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `service_id` INT NOT NULL,
    `type` ENUM('upgrade', 'downgrade', 'cancellation', 'cycle_change') NOT NULL,
    `data` JSON DEFAULT NULL,
    `invoice_id` INT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_service` (`service_id`),
    INDEX `idx_type` (`type`),
    INDEX `idx_created` (`created_at`)
);

-- Prorate credits
CREATE TABLE `mod_prorate_credits` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `service_id` INT NOT NULL,
    `client_id` INT NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `description` TEXT DEFAULT NULL,
    `applied_to_invoice` INT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `used_at` DATETIME DEFAULT NULL,
    INDEX `idx_client` (`client_id`),
    INDEX `idx_service` (`service_id`)
);

-- Custom product pricing for modules
CREATE TABLE `mod_product_pricing` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL UNIQUE,
    `monthly` DECIMAL(10,2) DEFAULT 0,
    `quarterly` DECIMAL(10,2) DEFAULT 0,
    `semiannual` DECIMAL(10,2) DEFAULT 0,
    `annual` DECIMAL(10,2) DEFAULT 0,
    `biennial` DECIMAL(10,2) DEFAULT 0,
    `setupfee` DECIMAL(10,2) DEFAULT 0,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ServiceChangePackage` | 1 | Calculate upgrade/downgrade prorate |
| `AfterServiceChangePackage` | 1 | Apply prorate changes |
| `ServiceTerminationRequest` | 1 | Calculate cancellation refund |
| `ServiceTerminated` | 1 | Create refund credit |
| `ServiceChangeBillingCycle` | 1 | Handle cycle change |
| `AdminAreaViewService` | 1 | Show prorate history |
| `BeforeServiceChange` | 1 | Validate for pending invoices |
| `ServiceChangeCompleted` | 1 | Update renewal amount |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement upgrade calculation
- [ ] Implement downgrade calculation
- [ ] Implement cancellation refund
- [ ] Implement cycle change calculation
- [ ] Create invoice generation
- [ ] Build admin interface
- [ ] Add history logging
- [ ] Create preview functionality
- [ ] Add credit handling
- [ ] Implement validation
- [ ] Test upgrade scenarios
- [ ] Test downgrade scenarios
- [ ] Verify cancellation refunds
- [ ] Test cycle changes
- [ ] Add reporting
- [ ] Implement edge cases
- [ ] Add email notifications
- [ ] Create documentation