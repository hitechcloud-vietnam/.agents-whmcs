# WHMCS Subscription Module DevKit

## Header

**Purpose:** Subscription management system providing flexible recurring billing options, subscription tiers, upgrade/downgrade handling, pause functionality, and subscription lifecycle management.

**Module Type:** Billing/Subscription Module

**Use Case:** Hosting companies offering SaaS-style subscriptions, tiered service plans, or subscription-based products requiring advanced management beyond standard WHMCS recurring billing.

---

## Complete Code Template

### File Structure
```
whmcs-subscription-module/
├── README.md
├── DEVKIT.md
├── subscription.php      # Main subscription logic
├── hooks.php             # WHMCS hook integrations
├── tier_manager.php      # Tier management
└── templates/
    └── admin_subscription.tpl
```

### Main Module File: subscription.php

```php
<?php
/**
 * WHMCS Subscription Module
 * 
 * Provides comprehensive subscription management.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('SUBSCRIPTION_MODULE_VERSION', '1.0.0');

// Subscription Status
define('SUB_STATUS_ACTIVE', 'active');
define('SUB_STATUS_PENDING', 'pending');
define('SUB_STATUS_PAUSED', 'paused');
define('SUB_STATUS_CANCELLED', 'cancelled');
define('SUB_STATUS_EXPIRED', 'expired');
define('SUB_STATUS_TRIAL', 'trial');

// Subscription Events
define('SUB_EVENT_CREATED', 'created');
define('SUB_EVENT_RENEWED', 'renewed');
define('SUB_EVENT_UPGRADED', 'upgraded');
define('SUB_EVENT_DOWNGRADED', 'downgraded');
define('SUB_EVENT_PAUSED', 'paused');
define('SUB_EVENT_RESUMED', 'resumed');
define('SUB_EVENT_CANCELLED', 'cancelled');

// Billing Cycles
define('BILLING_MONTHLY', 'monthly');
define('BILLING_QUARTERLY', 'quarterly');
define('BILLING_SEMIANNUAL', 'semiannual');
define('BILLING_ANNUAL', 'annual');
define('BILLING_BIENNIAL', 'biennial');
define('BILLING_TRIENNIAL', 'triennial');

/**
 * Create new subscription
 */
function subscription_create($clientId, $productId, $tierId, $data = []) {
    $subscriptionId = subscription_generate_id();
    
    $fields = [
        'subscription_id', 'client_id', 'product_id', 'tier_id',
        'status', 'billing_cycle', 'amount', 'trial_days', 'trial_end',
        'start_date', 'next_billing_date', 'end_date', 'paused_at',
        'pause_count', 'created_at'
    ];
    
    $product = get_product($productId);
    $tier = subscription_get_tier($tierId);
    
    $trialDays = $data['trial_days'] ?? $tier['trial_days'] ?? 0;
    $nextBilling = $trialDays > 0 
        ? date('Y-m-d', strtotime('+' . $trialDays . ' days'))
        : date('Y-m-d');
    
    $values = [
        $subscriptionId, $clientId, $productId, $tierId,
        $trialDays > 0 ? SUB_STATUS_TRIAL : SUB_STATUS_ACTIVE,
        $data['billing_cycle'] ?? $tier['billing_cycle'] ?? BILLING_MONTHLY,
        $data['amount'] ?? $tier['monthly_price'],
        $trialDays, $trialDays > 0 ? date('Y-m-d', strtotime('+' . $trialDays . ' days')) : null,
        date('Y-m-d'), $nextBilling, null, null, 0, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_subscriptions', array_combine($fields, $values));
    
    $subId = mysql_insert_id();
    
    // Log creation event
    subscription_log_event($subId, SUB_EVENT_CREATED, [
        'product_id' => $productId,
        'tier_id' => $tierId,
        'amount' => $tier['monthly_price']
    ]);
    
    return $subId;
}

/**
 * Generate unique subscription ID
 */
function subscription_generate_id() {
    return 'SUB-' . strtoupper(substr(md5(uniqid()), 0, 8)) . '-' . date('ymd');
}

/**
 * Get subscription by ID
 */
function subscription_get($subscriptionId) {
    $query = "SELECT s.*, c.firstname, c.lastname, c.email,
              p.name as product_name, t.name as tier_name
              FROM mod_subscriptions s
              JOIN tblclients c ON s.client_id = c.id
              LEFT JOIN tblproducts p ON s.product_id = p.id
              LEFT JOIN mod_subscription_tiers t ON s.tier_id = t.id
              WHERE s.id = ? OR s.subscription_id = ?";
    $result = full_query($query, [$subscriptionId, $subscriptionId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get subscription tier details
 */
function subscription_get_tier($tierId) {
    $query = "SELECT * FROM mod_subscription_tiers WHERE id = ?";
    $result = full_query($query, [$tierId]);
    return mysql_fetch_assoc($result);
}

/**
 * Process subscription renewal
 */
function subscription_process_renewal($subscriptionId) {
    $subscription = subscription_get($subscriptionId);
    
    if (!$subscription || $subscription['status'] != SUB_STATUS_ACTIVE) {
        return ['success' => false, 'error' => 'Subscription not eligible for renewal'];
    }
    
    // Calculate next billing date
    $nextDate = subscription_calculate_next_date(
        $subscription['next_billing_date'],
        $subscription['billing_cycle']
    );
    
    // Create invoice
    $invoiceId = subscription_create_invoice($subscription);
    
    // Update subscription
    update_query('mod_subscriptions', [
        'next_billing_date' => $nextDate,
        'last_renewal_at' => date('Y-m-d H:i:s'),
        'renewal_count' => $subscription['renewal_count'] + 1
    ], ['id' => $subscription['id']]);
    
    // Log event
    subscription_log_event($subscription['id'], SUB_EVENT_RENEWED, [
        'invoice_id' => $invoiceId,
        'amount' => $subscription['amount']
    ]);
    
    return [
        'success' => true,
        'invoice_id' => $invoiceId,
        'next_billing_date' => $nextDate
    ];
}

/**
 * Calculate next billing date based on cycle
 */
function subscription_calculate_next_date($currentDate, $cycle) {
    switch ($cycle) {
        case BILLING_MONTHLY:
            return date('Y-m-d', strtotime('+1 month', strtotime($currentDate)));
        case BILLING_QUARTERLY:
            return date('Y-m-d', strtotime('+3 months', strtotime($currentDate)));
        case BILLING_SEMIANNUAL:
            return date('Y-m-d', strtotime('+6 months', strtotime($currentDate)));
        case BILLING_ANNUAL:
            return date('Y-m-d', strtotime('+1 year', strtotime($currentDate)));
        case BILLING_BIENNIAL:
            return date('Y-m-d', strtotime('+2 years', strtotime($currentDate)));
        case BILLING_TRIENNIAL:
            return date('Y-m-d', strtotime('+3 years', strtotime($currentDate)));
        default:
            return date('Y-m-d', strtotime('+1 month', strtotime($currentDate)));
    }
}

/**
 * Create invoice for subscription
 */
function subscription_create_invoice($subscription) {
    global $CONFIG;
    
    $dueDate = date('Y-m-d', strtotime('+' . $CONFIG['InvoiceDueDays'] . ' days'));
    
    $invoiceData = [
        'userid' => $subscription['client_id'],
        'date' => date('Y-m-d'),
        'duedate' => $dueDate,
        'status' => 'Unpaid',
        'paymentmethod' => 'subscription',
        'notes' => 'Subscription renewal: ' . $subscription['subscription_id']
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    $itemData = [
        'invoiceid' => $invoiceId,
        'userid' => $subscription['client_id'],
        'description' => $subscription['product_name'] . ' - ' . $subscription['tier_name'] . ' (' . ucfirst($subscription['billing_cycle']) . ')',
        'amount' => $subscription['amount'],
        'taxed' => 0
    ];
    
    insert_query('tblinvoiceitems', $itemData);
    
    // Link invoice to subscription
    insert_query('mod_subscription_invoices', [
        'subscription_id' => $subscription['id'],
        'invoice_id' => $invoiceId
    ]);
    
    return $invoiceId;
}

/**
 * Pause subscription
 */
function subscription_pause($subscriptionId, $duration = null) {
    $subscription = subscription_get($subscriptionId);
    
    if (!$subscription || $subscription['status'] != SUB_STATUS_ACTIVE) {
        return ['success' => false, 'error' => 'Cannot pause subscription'];
    }
    
    $pauseEnd = $duration 
        ? date('Y-m-d', strtotime('+' . $duration . ' days'))
        : null;
    
    update_query('mod_subscriptions', [
        'status' => SUB_STATUS_PAUSED,
        'paused_at' => date('Y-m-d H:i:s'),
        'pause_end' => $pauseEnd,
        'pause_count' => $subscription['pause_count'] + 1
    ], ['id' => $subscription['id']]);
    
    subscription_log_event($subscription['id'], SUB_EVENT_PAUSED, [
        'pause_end' => $pauseEnd,
        'reason' => $pauseEnd ? 'Scheduled pause' : 'Manual pause'
    ]);
    
    return ['success' => true, 'pause_end' => $pauseEnd];
}

/**
 * Resume paused subscription
 */
function subscription_resume($subscriptionId) {
    $subscription = subscription_get($subscriptionId);
    
    if (!$subscription || $subscription['status'] != SUB_STATUS_PAUSED) {
        return ['success' => false, 'error' => 'Subscription is not paused'];
    }
    
    // Calculate new next billing date
    $nextDate = subscription_calculate_next_date(
        date('Y-m-d'),
        $subscription['billing_cycle']
    );
    
    update_query('mod_subscriptions', [
        'status' => SUB_STATUS_ACTIVE,
        'paused_at' => null,
        'pause_end' => null,
        'next_billing_date' => $nextDate
    ], ['id' => $subscription['id']]);
    
    subscription_log_event($subscription['id'], SUB_EVENT_RESUMED, [
        'new_next_date' => $nextDate
    ]);
    
    return ['success' => true, 'next_billing_date' => $nextDate];
}

/**
 * Cancel subscription
 */
function subscription_cancel($subscriptionId, $reason = '', $immediate = false) {
    $subscription = subscription_get($subscriptionId);
    
    if (!$subscription) {
        return ['success' => false, 'error' => 'Subscription not found'];
    }
    
    $endDate = $immediate 
        ? date('Y-m-d') 
        : $subscription['next_billing_date'];
    
    update_query('mod_subscriptions', [
        'status' => SUB_STATUS_CANCELLED,
        'end_date' => $endDate,
        'cancel_reason' => $reason,
        'cancelled_at' => date('Y-m-d H:i:s')
    ], ['id' => $subscription['id']]);
    
    subscription_log_event($subscription['id'], SUB_EVENT_CANCELLED, [
        'reason' => $reason,
        'immediate' => $immediate,
        'end_date' => $endDate
    ]);
    
    // If immediate cancellation, terminate associated service
    if ($immediate) {
        full_query("UPDATE tblhosting SET domainstatus = 'Cancelled' WHERE id = ?", 
                   [$subscription['service_id']]);
    }
    
    return ['success' => true, 'end_date' => $endDate];
}

/**
 * Upgrade/downgrade subscription tier
 */
function subscription_change_tier($subscriptionId, $newTierId, $prorate = true) {
    $subscription = subscription_get($subscriptionId);
    $newTier = subscription_get_tier($newTierId);
    
    if (!$subscription || !$newTier) {
        return ['success' => false, 'error' => 'Invalid subscription or tier'];
    }
    
    $oldTier = subscription_get_tier($subscription['tier_id']);
    $isUpgrade = $newTier['monthly_price'] > $oldTier['monthly_price'];
    
    // Calculate prorated amount if applicable
    $prorateAmount = 0;
    if ($prorate && $subscription['billing_cycle'] == BILLING_MONTHLY) {
        $daysRemaining = (strtotime($subscription['next_billing_date']) - time()) / 86400;
        $daysInPeriod = 30;
        
        $priceDiff = $newTier['monthly_price'] - $oldTier['monthly_price'];
        $prorateAmount = ($priceDiff * $daysRemaining) / $daysInPeriod;
    }
    
    // Update subscription tier
    update_query('mod_subscriptions', [
        'tier_id' => $newTierId,
        'amount' => $newTier['monthly_price']
    ], ['id' => $subscription['id']]);
    
    $event = $isUpgrade ? SUB_EVENT_UPGRADED : SUB_EVENT_DOWNGRADED;
    
    subscription_log_event($subscription['id'], $event, [
        'old_tier' => $oldTier['name'],
        'new_tier' => $newTier['name'],
        'old_price' => $oldTier['monthly_price'],
        'new_price' => $newTier['monthly_price'],
        'prorate_amount' => $prorateAmount
    ]);
    
    // Create prorate invoice if applicable
    if ($prorateAmount != 0) {
        subscription_create_prorate_invoice($subscription, $prorateAmount);
    }
    
    return [
        'success' => true,
        'is_upgrade' => $isUpgrade,
        'prorate_amount' => $prorateAmount,
        'new_amount' => $newTier['monthly_price']
    ];
}

/**
 * Create prorate invoice
 */
function subscription_create_prorate_invoice($subscription, $amount) {
    $invoiceData = [
        'userid' => $subscription['client_id'],
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d'),
        'status' => 'Unpaid',
        'notes' => 'Prorate adjustment: ' . $subscription['subscription_id']
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    $itemData = [
        'invoiceid' => $invoiceId,
        'userid' => $subscription['client_id'],
        'description' => 'Tier change prorate adjustment',
        'amount' => $amount,
        'taxed' => 0
    ];
    
    insert_query('tblinvoiceitems', $itemData);
    
    return $invoiceId;
}

/**
 * Log subscription event
 */
function subscription_log_event($subscriptionId, $event, $data = []) {
    $fields = ['subscription_id', 'event', 'data', 'created_at'];
    $values = [
        $subscriptionId, $event, json_encode($data), date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_subscription_events', array_combine($fields, $values));
}

/**
 * Get subscription events
 */
function subscription_get_events($subscriptionId, $limit = 50) {
    $query = "SELECT * FROM mod_subscription_events 
              WHERE subscription_id = ?
              ORDER BY created_at DESC LIMIT ?";
    $result = full_query($query, [$subscriptionId, $limit]);
    
    $events = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['data'] = json_decode($row['data'], true);
        $events[] = $row;
    }
    
    return $events;
}

/**
 * Get client subscriptions
 */
function subscription_get_client_subscriptions($clientId) {
    $query = "SELECT s.*, p.name as product_name, t.name as tier_name
              FROM mod_subscriptions s
              LEFT JOIN tblproducts p ON s.product_id = p.id
              LEFT JOIN mod_subscription_tiers t ON s.tier_id = t.id
              WHERE s.client_id = ?
              ORDER BY s.created_at DESC";
    $result = full_query($query, [$clientId]);
    
    $subscriptions = [];
    while ($row = mysql_fetch_assoc($result)) {
        $subscriptions[] = $row;
    }
    
    return $subscriptions;
}

/**
 * Create subscription tier
 */
function subscription_create_tier($data) {
    $fields = [
        'name', 'description', 'billing_cycle', 'monthly_price',
        'quarterly_price', 'semiannual_price', 'annual_price',
        'features', 'trial_days', 'max_products', 'sort_order',
        'status', 'created_at'
    ];
    
    $values = [
        $data['name'], $data['description'] ?? '', $data['billing_cycle'] ?? BILLING_MONTHLY,
        $data['monthly_price'], $data['quarterly_price'] ?? 0,
        $data['semiannual_price'] ?? 0, $data['annual_price'] ?? 0,
        json_encode($data['features'] ?? []), $data['trial_days'] ?? 0,
        $data['max_products'] ?? 1, $data['sort_order'] ?? 0,
        $data['status'] ?? 'active', date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_subscription_tiers', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Get all subscription tiers
 */
function subscription_get_all_tiers($productId = null) {
    $where = "";
    $params = [];
    
    if ($productId) {
        $where = "WHERE product_id = ?";
        $params[] = $productId;
    }
    
    $query = "SELECT * FROM mod_subscription_tiers {$where} ORDER BY sort_order ASC";
    $result = full_query($query, $params);
    
    $tiers = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['features'] = json_decode($row['features'], true);
        $tiers[] = $row;
    }
    
    return $tiers;
}

/**
 * Process trial expiration
 */
function subscription_process_trial($subscriptionId) {
    $subscription = subscription_get($subscriptionId);
    
    if (!$subscription || $subscription['status'] != SUB_STATUS_TRIAL) {
        return ['success' => false];
    }
    
    // Check if trial ended
    if ($subscription['trial_end'] && $subscription['trial_end'] <= date('Y-m-d')) {
        // Convert to active and create first invoice
        update_query('mod_subscriptions', [
            'status' => SUB_STATUS_ACTIVE,
            'trial_end' => null
        ], ['id' => $subscription['id']]);
        
        $invoiceId = subscription_create_invoice($subscription);
        
        subscription_log_event($subscription['id'], SUB_EVENT_RENEWED, [
            'trial_converted' => true,
            'first_invoice_id' => $invoiceId
        ]);
        
        return ['success' => true, 'invoice_id' => $invoiceId];
    }
    
    return ['success' => false, 'message' => 'Trial not yet ended'];
}

/**
 * Get subscription statistics
 */
function subscription_get_stats($filters = []) {
    $where = "1=1";
    $params = [];
    
    if (!empty($filters['status'])) {
        $where .= " AND s.status = ?";
        $params[] = $filters['status'];
    }
    
    if (!empty($filters['start_date'])) {
        $where .= " AND s.created_at >= ?";
        $params[] = $filters['start_date'];
    }
    
    if (!empty($filters['end_date'])) {
        $where .= " AND s.created_at <= ?";
        $params[] = $filters['end_date'];
    }
    
    $query = "SELECT 
                COUNT(*) as total_subscriptions,
                SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN status = 'paused' THEN 1 ELSE 0 END) as paused,
                SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
                SUM(CASE WHEN status = 'trial' THEN 1 ELSE 0 END) as trial,
                SUM(CASE WHEN status = 'active' THEN amount ELSE 0 END) as mrr,
                COUNT(DISTINCT client_id) as unique_clients
              FROM mod_subscriptions s
              WHERE {$where}";
    
    $result = full_query($query, $params);
    return mysql_fetch_assoc($result);
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Subscription Module Hooks
 */

// Hook: Create subscription on service creation
add_hook('ServiceCreated', 1, function($params) {
    $serviceId = $params['serviceId'];
    $clientId = $params['userId'];
    $productId = $params['productId'];
    
    // Check if product supports subscriptions
    if (!subscription_product_enabled($productId)) {
        return [];
    }
    
    // Get default tier for product
    $tierId = subscription_get_default_tier($productId);
    
    if ($tierId) {
        $subId = subscription_create($clientId, $productId, $tierId, [
            'service_id' => $serviceId
        ]);
        
        // Link to service
        update_query('mod_subscriptions', ['service_id' => $serviceId], ['id' => $subId]);
        
        return ['subscription_id' => $subId];
    }
});

// Hook: Process subscription renewal
add_hook('DailyCronJob', 1, function($params) {
    // Find due renewals
    $query = "SELECT id FROM mod_subscriptions 
              WHERE status = 'active' 
              AND next_billing_date <= CURDATE()";
    $result = full_query($query);
    
    $processed = 0;
    while ($row = mysql_fetch_assoc($result)) {
        $result = subscription_process_renewal($row['id']);
        if ($result['success']) {
            $processed++;
        }
    }
    
    return ['processed' => $processed];
});

// Hook: Process trial expirations
add_hook('DailyCronJob', 2, function($params) {
    $query = "SELECT id FROM mod_subscriptions WHERE status = 'trial' AND trial_end <= CURDATE()";
    $result = full_query($query);
    
    while ($row = mysql_fetch_assoc($result)) {
        subscription_process_trial($row['id']);
    }
});

// Hook: Resume paused subscriptions
add_hook('DailyCronJob', 3, function($params) {
    $query = "SELECT id FROM mod_subscriptions 
              WHERE status = 'paused' AND pause_end IS NOT NULL AND pause_end <= CURDATE()";
    $result = full_query($query);
    
    while ($row = mysql_fetch_assoc($result)) {
        subscription_resume($row['id']);
    }
});

// Hook: Handle subscription cancellation
add_hook('ServiceTermination', 1, function($params) {
    $serviceId = $params['serviceId'];
    
    // Find and cancel associated subscription
    $query = "SELECT id FROM mod_subscriptions WHERE service_id = ? AND status IN ('active', 'trial')";
    $result = full_query($query, [$serviceId]);
    
    while ($row = mysql_fetch_assoc($result)) {
        subscription_cancel($row['id'], 'Service terminated', true);
    }
});

// Hook: Apply subscription discount
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check if this is a subscription renewal
    $query = "SELECT si.subscription_id, s.amount, s.billing_cycle
              FROM mod_subscription_invoices si
              JOIN mod_subscriptions s ON si.subscription_id = s.id
              WHERE si.invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    if ($subInvoice = mysql_fetch_assoc($result)) {
        // Apply any subscription-level discounts
        $discount = subscription_calculate_discount($subInvoice['subscription_id']);
        
        if ($discount > 0) {
            addInvoiceItem($invoiceId, 0, 'Subscription Discount', 
                          'Loyalty discount', -$discount, 0);
        }
    }
});

// Hook: Track subscription changes
add_hook('AfterShoppingCartValidate', 1, function($params) {
    // Check for tier change in cart
    if (!empty($_SESSION['cart']['subscription_upgrade'])) {
        $oldTierId = $_SESSION['cart']['current_tier_id'];
        $newTierId = $_SESSION['cart']['subscription_upgrade'];
        
        $prorateAmount = subscription_calculate_prorate($oldTierId, $newTierId);
        
        return [
            'prorate_required' => true,
            'prorate_amount' => $prorateAmount
        ];
    }
});

// Hook: Update subscription on payment
add_hook('InvoicePaid', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check subscription link
    $query = "SELECT subscription_id FROM mod_subscription_invoices WHERE invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    if ($row = mysql_fetch_assoc($result)) {
        // Update last payment info
        update_query('mod_subscriptions', [
            'last_payment_at' => date('Y-m-d H:i:s'),
            'last_payment_amount' => getInvoiceTotal($invoiceId)
        ], ['id' => $row['subscription_id']]);
    }
});
```

---

## Database Schema

```sql
-- Main subscription table
CREATE TABLE `mod_subscriptions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `subscription_id` VARCHAR(50) NOT NULL UNIQUE,
    `client_id` INT NOT NULL,
    `product_id` INT NOT NULL,
    `tier_id` INT NOT NULL,
    `service_id` INT DEFAULT NULL,
    `status` ENUM('active', 'pending', 'paused', 'cancelled', 'expired', 'trial') DEFAULT 'active',
    `billing_cycle` ENUM('monthly', 'quarterly', 'semiannual', 'annual', 'biennial', 'triennial') DEFAULT 'monthly',
    `amount` DECIMAL(10,2) NOT NULL,
    `trial_days` INT DEFAULT 0,
    `trial_end` DATE DEFAULT NULL,
    `start_date` DATE DEFAULT NULL,
    `next_billing_date` DATE DEFAULT NULL,
    `end_date` DATE DEFAULT NULL,
    `paused_at` DATETIME DEFAULT NULL,
    `pause_end` DATE DEFAULT NULL,
    `pause_count` INT DEFAULT 0,
    `cancel_reason` TEXT DEFAULT NULL,
    `cancelled_at` DATETIME DEFAULT NULL,
    `last_payment_at` DATETIME DEFAULT NULL,
    `last_payment_amount` DECIMAL(10,2) DEFAULT NULL,
    `renewal_count` INT DEFAULT 0,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_client` (`client_id`),
    INDEX `idx_status` (`status`),
    INDEX `idx_next_billing` (`next_billing_date`),
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Subscription tiers
CREATE TABLE `mod_subscription_tiers` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT DEFAULT NULL,
    `name` VARCHAR(255) NOT NULL,
    `description` TEXT DEFAULT NULL,
    `billing_cycle` ENUM('monthly', 'quarterly', 'semiannual', 'annual') DEFAULT 'monthly',
    `monthly_price` DECIMAL(10,2) NOT NULL,
    `quarterly_price` DECIMAL(10,2) DEFAULT NULL,
    `semiannual_price` DECIMAL(10,2) DEFAULT NULL,
    `annual_price` DECIMAL(10,2) DEFAULT NULL,
    `features` JSON DEFAULT NULL,
    `trial_days` INT DEFAULT 0,
    `max_products` INT DEFAULT 1,
    `sort_order` INT DEFAULT 0,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Subscription events log
CREATE TABLE `mod_subscription_events` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `subscription_id` INT NOT NULL,
    `event` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_subscription` (`subscription_id`),
    INDEX `idx_event` (`event`),
    FOREIGN KEY (`subscription_id`) REFERENCES `mod_subscriptions`(`id`) ON DELETE CASCADE
);

-- Subscription-invoice linking
CREATE TABLE `mod_subscription_invoices` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `subscription_id` INT NOT NULL,
    `invoice_id` INT NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_invoice` (`invoice_id`),
    FOREIGN KEY (`subscription_id`) REFERENCES `mod_subscriptions`(`id`) ON DELETE CASCADE,
    FOREIGN KEY (`invoice_id`) REFERENCES `tblinvoices`(`id`) ON DELETE CASCADE
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ServiceCreated` | 1 | Create subscription on service creation |
| `DailyCronJob` | 1 | Process subscription renewals |
| `DailyCronJob` | 2 | Process trial expirations |
| `DailyCronJob` | 3 | Resume paused subscriptions |
| `ServiceTermination` | 1 | Cancel subscription on termination |
| `InvoiceCreationPreCheck` | 1 | Apply subscription discounts |
| `AfterShoppingCartValidate` | 1 | Track tier changes |
| `InvoicePaid` | 1 | Update payment info |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement subscription creation
- [ ] Create tier management
- [ ] Implement renewal processing
- [ ] Add pause/resume functionality
- [ ] Create upgrade/downgrade logic
- [ ] Implement prorate calculations
- [ ] Add trial period handling
- [ ] Create cancellation flow
- [ ] Build admin interface
- [ ] Add subscription events logging
- [ ] Implement analytics dashboard
- [ ] Create customer portal views
- [ ] Add email notifications
- [ ] Test billing cycle calculations
- [ ] Verify trial conversion
- [ ] Test tier changes
- [ ] Add webhook integrations
- [ ] Implement subscription reports