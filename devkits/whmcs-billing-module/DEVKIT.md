# WHMCS Billing Module DevKit

## Header

**Purpose:** Custom billing rules engine that handles complex billing scenarios including custom invoice generation, billing cycles, and payment scheduling based on configurable business rules.

**Module Type:** Billing/Automation Module

**Use Case:** Hosting providers needing custom billing logic beyond standard WHMCS, such as milestone-based billing, usage-based charges, or custom payment schedules.

---

## Complete Code Template

### File Structure
```
whmcs-billing-module/
├── README.md
├── DEVKIT.md
├── billing.php           # Main billing logic
├── hooks.php             # WHMCS hook integrations
├── rules.php             # Billing rule engine
└── templates/
    └── admin_billing.tpl
```

### Main Module File: billing.php

```php
<?php
/**
 * WHMCS Custom Billing Module
 * 
 * Provides configurable billing rules engine for complex scenarios.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('BILLING_MODULE_VERSION', '1.0.0');

// Billing Rule Types
define('BILLING_RULE_FIXED', 'fixed');
define('BILLING_RULE_USAGE', 'usage');
define('BILLING_RULE_MILESTONE', 'milestone');
define('BILLING_RULE_RECURRING', 'recurring');

// Billing Status
define('BILLING_STATUS_PENDING', 'pending');
define('BILLING_STATUS_ACTIVE', 'active');
define('BILLING_STATUS_COMPLETED', 'completed');
define('BILLING_STATUS_CANCELLED', 'cancelled');

// Milestone Status
define('MILESTONE_PENDING', 'pending');
define('MILESTONE_INVOICED', 'invoiced');
define('MILESTONE_PAID', 'paid');
define('MILESTONE_SKIPPED', 'skipped');

/**
 * Process billing for an order
 */
function billing_process_order($orderId, $rules = []) {
    $orderData = billing_get_order_data($orderId);
    
    if (!$orderData) {
        return ['success' => false, 'error' => 'Order not found'];
    }
    
    $invoiceId = billing_create_invoice($orderData, $rules);
    
    return [
        'success' => true,
        'invoice_id' => $invoiceId,
        'total_amount' => $orderData['total']
    ];
}

/**
 * Get order data for billing
 */
function billing_get_order_data($orderId) {
    $query = "SELECT o.*, c.firstname, c.lastname, c.email, c.companyname,
              h.domain, h.domainreg
              FROM tblorders o
              JOIN tblclients c ON o.userid = c.id
              LEFT JOIN tblhosting h ON o.id = h.orderid
              WHERE o.id = ?";
    $result = full_query($query, [$orderId]);
    $order = mysql_fetch_assoc($result);
    
    if (!$order) {
        return null;
    }
    
    // Get order items
    $order['items'] = billing_get_order_items($orderId);
    
    // Calculate total
    $order['total'] = array_sum(array_column($order['items'], 'amount'));
    
    return $order;
}

/**
 * Get all items in an order
 */
function billing_get_order_items($orderId) {
    $query = "SELECT oi.*, p.name as product_name, p.description
              FROM tblorderitems oi
              JOIN tblproducts p ON oi.relid = p.id
              WHERE oi.orderid = ?";
    $result = full_query($query, [$orderId]);
    
    $items = [];
    while ($row = mysql_fetch_assoc($result)) {
        $items[] = $row;
    }
    
    return $items;
}

/**
 * Create invoice with billing rules applied
 */
function billing_create_invoice($orderData, $rules) {
    global $CONFIG;
    
    $dueDate = date('Y-m-d', strtotime('+' . $CONFIG['InvoiceDueDays'] . ' days'));
    
    $invoiceData = [
        'userid' => $orderData['userid'],
        'invoicenum' => billing_generate_invoice_number(),
        'date' => date('Y-m-d'),
        'duedate' => $dueDate,
        'datepaid' => null,
        'status' => 'Unpaid',
        'paymentmethod' => $orderData['paymentmethod'] ?? 'banktransfer',
        'notes' => 'Custom billing invoice for Order #' . $orderData['id']
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    // Add line items
    foreach ($orderData['items'] as $item) {
        billing_add_invoice_item($invoiceId, $item, $rules);
    }
    
    // Apply any additional charges
    billing_apply_additional_charges($invoiceId, $orderData, $rules);
    
    return $invoiceId;
}

/**
 * Generate custom invoice number
 */
function billing_generate_invoice_number() {
    $prefix = 'BILL-' . date('Ym');
    $query = "SELECT MAX(CAST(SUBSTRING(invoicenum, " . (strlen($prefix) + 1) . ") AS UNSIGNED)) as max_num 
              FROM tblinvoices WHERE invoicenum LIKE ?";
    $result = full_query($query, [$prefix . '%']);
    $data = mysql_fetch_assoc($result);
    
    $nextNum = ($data['max_num'] ?? 0) + 1;
    return $prefix . '-' . str_pad($nextNum, 4, '0', STR_PAD_LEFT);
}

/**
 * Add item to invoice with rules applied
 */
function billing_add_invoice_item($invoiceId, $item, $rules) {
    $amount = $item['amount'];
    
    // Apply custom pricing rules
    if (!empty($rules)) {
        foreach ($rules as $rule) {
            if ($rule['product_id'] == $item['relid']) {
                $amount = billing_apply_rule($amount, $rule);
            }
        }
    }
    
    $itemData = [
        'invoiceid' => $invoiceId,
        'userid' => $item['userid'] ?? 0,
        'description' => $item['description'] ?: $item['product_name'],
        'amount' => $amount,
        'taxed' => $item['taxed'] ?? 0
    ];
    
    insert_query('tblinvoiceitems', $itemData);
}

/**
 * Apply billing rule to amount
 */
function billing_apply_rule($amount, $rule) {
    switch ($rule['type']) {
        case BILLING_RULE_FIXED:
            return $rule['value'];
            
        case BILLING_RULE_PERCENTAGE:
            return $amount * (1 - ($rule['value'] / 100));
            
        case BILLING_RULE_MILESTONE:
            return $amount * ($rule['percentage'] / 100);
            
        default:
            return $amount;
    }
}

/**
 * Apply additional charges (setup fees, etc.)
 */
function billing_apply_additional_charges($invoiceId, $orderData, $rules) {
    $charges = [];
    
    // Check for setup fee rules
    foreach ($rules as $rule) {
        if ($rule['type'] === 'setup_fee') {
            $charges[] = [
                'description' => $rule['name'],
                'amount' => $rule['value'],
                'taxed' => $rule['taxable'] ?? false
            ];
        }
    }
    
    // Add milestone charges
    $milestones = billing_get_pending_milestones($orderData['id']);
    foreach ($milestones as $milestone) {
        if ($milestone['trigger'] === 'order_placed') {
            $charges[] = [
                'description' => $milestone['name'],
                'amount' => $milestone['amount'],
                'taxed' => $milestone['taxable']
            ];
        }
    }
    
    foreach ($charges as $charge) {
        $itemData = [
            'invoiceid' => $invoiceId,
            'userid' => $orderData['userid'],
            'description' => $charge['description'],
            'amount' => $charge['amount'],
            'taxed' => $charge['taxed'] ? 1 : 0
        ];
        insert_query('tblinvoiceitems', $itemData);
    }
}

/**
 * Get pending milestones for an order
 */
function billing_get_pending_milestones($orderId) {
    $query = "SELECT * FROM mod_billing_milestones 
              WHERE order_id = ? AND status = ?";
    $result = full_query($query, [$orderId, MILESTONE_PENDING]);
    
    $milestones = [];
    while ($row = mysql_fetch_assoc($result)) {
        $milestones[] = $row;
    }
    
    return $milestones;
}

/**
 * Create billing rule
 */
function billing_create_rule($data) {
    $fields = ['name', 'type', 'product_id', 'value', 'percentage', 
               'conditions', 'priority', 'status', 'created_at'];
    
    $values = [
        $data['name'], $data['type'], $data['product_id'] ?? 0,
        $data['value'] ?? 0, $data['percentage'] ?? 100,
        json_encode($data['conditions'] ?? []), $data['priority'] ?? 50,
        $data['status'] ?? BILLING_STATUS_ACTIVE, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_billing_rules', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Create milestone billing schedule
 */
function billing_create_milestone($data) {
    $fields = ['order_id', 'name', 'amount', 'percentage', 'due_date', 
               'trigger', 'status', 'taxable', 'created_at'];
    
    $values = [
        $data['order_id'], $data['name'], $data['amount'],
        $data['percentage'] ?? 100, $data['due_date'], $data['trigger'],
        MILESTONE_PENDING, $data['taxable'] ?? false, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_billing_milestones', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Process milestone completion
 */
function billing_process_milestone($milestoneId, $invoiceId = null) {
    $updateData = [
        'status' => MILESTONE_INVOICED,
        'invoice_id' => $invoiceId,
        'invoiced_at' => date('Y-m-d H:i:s')
    ];
    
    update_query('mod_billing_milestones', $updateData, ['id' => $milestoneId]);
    
    // Trigger callback if set
    $milestone = billing_get_milestone($milestoneId);
    if (!empty($milestone['callback_url'])) {
        billing_trigger_callback($milestone);
    }
}

/**
 * Get milestone details
 */
function billing_get_milestone($milestoneId) {
    $query = "SELECT m.*, o.userid FROM mod_billing_milestones m
              JOIN tblorders o ON m.order_id = o.id
              WHERE m.id = ?";
    $result = full_query($query, [$milestoneId]);
    return mysql_fetch_assoc($result);
}

/**
 * Calculate usage-based billing
 */
function billing_calculate_usage($clientId, $productId, $period = 'monthly') {
    $usage = billing_get_usage_data($clientId, $productId, $period);
    $rates = billing_get_usage_rates($productId);
    
    $total = 0;
    foreach ($usage as $metric => $value) {
        if (isset($rates[$metric])) {
            $total += $value * $rates[$metric]['rate'];
            
            // Apply tiered pricing
            if (!empty($rates[$metric]['tiers'])) {
                $total += billing_calculate_tiered_cost($value, $rates[$metric]['tiers']);
            }
        }
    }
    
    return $total;
}

/**
 * Get usage data for billing period
 */
function billing_get_usage_data($clientId, $productId, $period) {
    $startDate = billing_get_period_start($period);
    $endDate = date('Y-m-d H:i:s');
    
    $query = "SELECT metric, SUM(value) as total 
              FROM mod_billing_usage 
              WHERE client_id = ? AND product_id = ?
              AND recorded_at BETWEEN ? AND ?
              GROUP BY metric";
    $result = full_query($query, [$clientId, $productId, $startDate, $endDate]);
    
    $usage = [];
    while ($row = mysql_fetch_assoc($result)) {
        $usage[$row['metric']] = $row['total'];
    }
    
    return $usage;
}

/**
 * Get usage rates for product
 */
function billing_get_usage_rates($productId) {
    $query = "SELECT * FROM mod_billing_usage_rates WHERE product_id = ?";
    $result = full_query($query, [$productId]);
    
    $rates = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rates[$row['metric']] = [
            'rate' => $row['rate'],
            'tiers' => json_decode($row['tiers_json'], true) ?: []
        ];
    }
    
    return $rates;
}

/**
 * Calculate tiered cost
 */
function billing_calculate_tiered_cost($quantity, $tiers) {
    $cost = 0;
    $remaining = $quantity;
    
    usort($tiers, function($a, $b) {
        return $a['min'] - $b['min'];
    });
    
    foreach ($tiers as $tier) {
        if ($remaining <= 0) break;
        
        $tierQty = min($remaining, $tier['max'] - $tier['min'] + 1);
        $cost += $tierQty * $tier['rate'];
        $remaining -= $tierQty;
    }
    
    return $cost;
}

/**
 * Get billing period start date
 */
function billing_get_period_start($period) {
    switch ($period) {
        case 'daily':
            return date('Y-m-d 00:00:00');
        case 'weekly':
            return date('Y-m-d 00:00:00', strtotime('-1 week'));
        case 'monthly':
            return date('Y-m-01 00:00:00');
        case 'quarterly':
            $quarter = floor((date('n') - 1) / 3);
            return date('Y-' . ($quarter * 3 + 1) . '-01 00:00:00');
        case 'annually':
            return date('Y-01-01 00:00:00');
        default:
            return date('Y-m-01 00:00:00');
    }
}

/**
 * Trigger webhook callback
 */
function billing_trigger_callback($milestone) {
    $payload = [
        'event' => 'milestone_invoiced',
        'milestone_id' => $milestone['id'],
        'order_id' => $milestone['order_id'],
        'client_id' => $milestone['userid'],
        'amount' => $milestone['amount'],
        'timestamp' => time()
    ];
    
    $ch = curl_init($milestone['callback_url']);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 10);
    curl_exec($ch);
    curl_close($ch);
}

/**
 * Get all billing rules
 */
function billing_get_all_rules($filters = []) {
    $where = "1=1";
    $params = [];
    
    if (!empty($filters['type'])) {
        $where .= " AND type = ?";
        $params[] = $filters['type'];
    }
    
    if (!empty($filters['status'])) {
        $where .= " AND status = ?";
        $params[] = $filters['status'];
    }
    
    $query = "SELECT * FROM mod_billing_rules WHERE {$where} ORDER BY priority DESC";
    $result = full_query($query, $params);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['conditions'] = json_decode($row['conditions'], true);
        $rules[] = $row;
    }
    
    return $rules;
}

/**
 * Validate billing rule
 */
function billing_validate_rule($data, &$errors) {
    if (empty($data['name'])) {
        $errors[] = "Rule name is required";
    }
    
    if (empty($data['type'])) {
        $errors[] = "Rule type is required";
    }
    
    if ($data['type'] === BILLING_RULE_FIXED && empty($data['value'])) {
        $errors[] = "Fixed value is required for fixed type rules";
    }
    
    if ($data['type'] === BILLING_RULE_MILESTONE && empty($data['percentage'])) {
        $errors[] = "Percentage is required for milestone rules";
    }
    
    return count($errors) === 0;
}

/**
 * Generate billing report
 */
function billing_generate_report($startDate, $endDate, $groupBy = 'daily') {
    $query = "SELECT 
                DATE(created_at) as date,
                COUNT(*) as total_milestones,
                SUM(amount) as total_amount,
                COUNT(CASE WHEN status = 'paid' THEN 1 END) as paid_count,
                SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END) as paid_amount
              FROM mod_billing_milestones
              WHERE created_at BETWEEN ? AND ?
              GROUP BY DATE(created_at)
              ORDER BY date DESC";
    
    $result = full_query($query, [$startDate, $endDate]);
    
    $report = [];
    while ($row = mysql_fetch_assoc($result)) {
        $report[] = $row;
    }
    
    return $report;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Billing Module Hooks
 */

// Hook: Process custom billing on order completion
add_hook('OrderCompleted', 1, function($params) {
    $orderId = $params['orderId'];
    
    // Get billing rules for this order
    $rules = billing_get_all_rules(['status' => 'active', 'type' => 'order']);
    
    // Process billing
    $result = billing_process_order($orderId, $rules);
    
    // Log the billing action
    logActivity("Custom billing processed for order #{$orderId}: Invoice #{$result['invoice_id']}");
    
    return $result;
});

// Hook: Apply milestone billing on service creation
add_hook('ServiceCreated', 1, function($params) {
    $orderId = $params['orderId'];
    
    // Create milestone schedule if applicable
    $milestones = billing_get_order_milestones($orderId);
    
    foreach ($milestones as $milestone) {
        billing_create_milestone([
            'order_id' => $orderId,
            'name' => $milestone['name'],
            'amount' => $milestone['amount'],
            'percentage' => $milestone['percentage'],
            'due_date' => $milestone['due_date'],
            'trigger' => $milestone['trigger']
        ]);
    }
});

// Hook: Calculate usage-based billing
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check if usage-based billing applies
    $items = get_query_vals("tblinvoiceitems", "*", ["invoiceid" => $invoiceId]);
    
    foreach ($items as $item) {
        if (billing_is_usage_product($item['relid'])) {
            $usageCharge = billing_calculate_usage(
                $item['userid'], 
                $item['relid'], 
                'monthly'
            );
            
            // Add usage item to invoice
            addInvoiceItem($invoiceId, $item['userid'], 'Usage', 
                          "Usage charges for " . date('F Y'), $usageCharge, 0);
        }
    }
});

// Hook: Handle milestone triggers
add_hook('InvoicePaid', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check for milestone triggers
    $query = "SELECT m.* FROM mod_billing_milestones m
              JOIN tblorders o ON m.order_id = o.id
              JOIN tblinvoices i ON i.userid = o.userid
              WHERE i.id = ? AND m.status = ?";
    $result = full_query($query, [$invoiceId, MILESTONE_PENDING]);
    
    while ($milestone = mysql_fetch_assoc($result)) {
        if ($milestone['trigger'] === 'payment_received') {
            billing_process_milestone($milestone['id'], $invoiceId);
            
            // Trigger next milestone
            billing_activate_next_milestone($milestone['order_id']);
        }
    }
});

// Hook: Cancel billing on order cancellation
add_hook('OrderCancelled', 1, function($params) {
    $orderId = $params['orderId'];
    
    // Update pending milestones
    update_query('mod_billing_milestones', 
                 ['status' => MILESTONE_SKIPPED], 
                 ['order_id' => $orderId, 'status' => MILESTONE_PENDING]);
});

// Hook: Modify invoice before display
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Add billing info to invoice view
    $milestones = billing_get_invoice_milestones($invoiceId);
    
    return [
        'billing_milestones' => $milestones,
        'custom_billing_info' => billing_get_invoice_billing_info($invoiceId)
    ];
});
```

### Admin Template: templates/admin_billing.tpl

```html
{extends file="admin/template.tpl"}

{block name="content"}
<div class="billing-admin">
    <div class="header">
        <h2>Custom Billing Configuration</h2>
        <div class="actions">
            <button class="btn btn-primary" onclick="showAddRuleModal()">Add Billing Rule</button>
            <button class="btn btn-secondary" onclick="showMilestoneModal()">Create Milestone</button>
        </div>
    </div>

    <div class="billing-tabs">
        <button class="tab active" data-tab="rules">Billing Rules</button>
        <button class="tab" data-tab="milestones">Milestones</button>
        <button class="tab" data-tab="usage">Usage Billing</button>
        <button class="tab" data-tab="reports">Reports</button>
    </div>

    <div id="tab-rules" class="tab-content">
        <table class="billing-table">
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Type</th>
                    <th>Value</th>
                    <th>Conditions</th>
                    <th>Status</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {foreach from=$rules item=rule}
                <tr>
                    <td>{$rule.name}</td>
                    <td><span class="badge badge-{$rule.type}">{$rule.type}</span></td>
                    <td>
                        {if $rule.type == 'percentage'}
                            {$rule.percentage}%
                        {elseif $rule.type == 'fixed'}
                            {$currency_prefix}{$rule.value}{$currency_suffix}
                        {else}
                            --
                        {/if}
                    </td>
                    <td>
                        {if $rule.conditions}
                            <span class="conditions-count">{count($rule.conditions)} conditions</span>
                        {else}
                            No conditions
                        {/if}
                    </td>
                    <td>
                        <span class="status-badge status-{$rule.status}">{$rule.status}</span>
                    </td>
                    <td>
                        <button onclick="editRule({$rule.id})">Edit</button>
                        <button onclick="deleteRule({$rule.id})">Delete</button>
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>

    <div id="tab-milestones" class="tab-content" style="display:none;">
        <table class="billing-table">
            <thead>
                <tr>
                    <th>Order</th>
                    <th>Milestone Name</th>
                    <th>Amount</th>
                    <th>Due Date</th>
                    <th>Trigger</th>
                    <th>Status</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {foreach from=$milestones item=m}
                <tr>
                    <td>#{$m.order_id}</td>
                    <td>{$m.name}</td>
                    <td>{$currency_prefix}{$m.amount}{$currency_suffix}</td>
                    <td>{$m.due_date}</td>
                    <td>{$m.trigger}</td>
                    <td><span class="status-badge status-{$m.status}">{$m.status}</span></td>
                    <td>
                        {if $m.status == 'pending'}
                            <button onclick="invoiceMilestone({$m.id})">Invoice</button>
                        {/if}
                        <button onclick="skipMilestone({$m.id})">Skip</button>
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>

    <div id="tab-usage" class="tab-content" style="display:none;">
        <form class="usage-form">
            <div class="form-row">
                <select name="client_id">
                    <option value="">Select Client</option>
                    {foreach from=$clients item=c}
                    <option value="{$c.id}">{$c.firstname} {$c.lastname}</option>
                    {/foreach}
                </select>
                <select name="product_id">
                    <option value="">Select Product</option>
                    {foreach from=$products item=p}
                    <option value="{$p.id}">{$p.name}</option>
                    {/foreach}
                </select>
                <select name="period">
                    <option value="monthly">Monthly</option>
                    <option value="quarterly">Quarterly</option>
                    <option value="annually">Annually</option>
                </select>
                <button type="button" class="btn" onclick="calculateUsage()">Calculate Usage</button>
            </div>
        </form>
        <div id="usage-results" class="usage-results"></div>
    </div>

    <div id="tab-reports" class="tab-content" style="display:none;">
        <form class="report-form">
            <div class="form-row">
                <input type="date" name="start_date" value="{$start_date}">
                <input type="date" name="end_date" value="{$end_date}">
                <button type="button" class="btn" onclick="generateReport()">Generate Report</button>
            </div>
        </form>
        <div id="report-results" class="report-results"></div>
    </div>
</div>
{/block}
```

---

## Database Schema

```sql
-- Billing rules table
CREATE TABLE `mod_billing_rules` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `type` ENUM('fixed', 'percentage', 'milestone', 'usage', 'recurring') NOT NULL,
    `product_id` INT DEFAULT NULL,
    `value` DECIMAL(10,2) DEFAULT NULL,
    `percentage` DECIMAL(5,2) DEFAULT 100,
    `conditions` JSON DEFAULT NULL,
    `priority` INT DEFAULT 50,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_type_status` (`type`, `status`),
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE SET NULL
);

-- Milestone billing schedule
CREATE TABLE `mod_billing_milestones` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `order_id` INT NOT NULL,
    `name` VARCHAR(255) NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `percentage` DECIMAL(5,2) DEFAULT 100,
    `due_date` DATE NOT NULL,
    `trigger` VARCHAR(50) DEFAULT NULL,
    `status` ENUM('pending', 'invoiced', 'paid', 'skipped') DEFAULT 'pending',
    `invoice_id` INT DEFAULT NULL,
    `taxable` TINYINT(1) DEFAULT 0,
    `callback_url` VARCHAR(500) DEFAULT NULL,
    `invoiced_at` DATETIME DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_order_status` (`order_id`, `status`),
    INDEX `idx_due_date` (`due_date`),
    FOREIGN KEY (`order_id`) REFERENCES `tblorders`(`id`) ON DELETE CASCADE,
    FOREIGN KEY (`invoice_id`) REFERENCES `tblinvoices`(`id`) ON DELETE SET NULL
);

-- Usage tracking
CREATE TABLE `mod_billing_usage` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `product_id` INT NOT NULL,
    `metric` VARCHAR(100) NOT NULL,
    `value` DECIMAL(15,4) NOT NULL,
    `unit` VARCHAR(20) DEFAULT NULL,
    `recorded_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_client_metric_date` (`client_id`, `metric`, `recorded_at`)
);

-- Usage rates configuration
CREATE TABLE `mod_billing_usage_rates` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL,
    `metric` VARCHAR(100) NOT NULL,
    `rate` DECIMAL(10,4) NOT NULL,
    `tiers_json` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE CASCADE,
    UNIQUE KEY `idx_product_metric` (`product_id`, `metric`)
);

-- Billing history log
CREATE TABLE `mod_billing_history` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `order_id` INT NOT NULL,
    `invoice_id` INT DEFAULT NULL,
    `action` VARCHAR(100) NOT NULL,
    `amount` DECIMAL(10,2) DEFAULT NULL,
    `details` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_order_date` (`order_id`, `created_at`)
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `OrderCompleted` | 1 | Process custom billing on order completion |
| `ServiceCreated` | 1 | Create milestone schedule for new services |
| `InvoiceCreationPreCheck` | 1 | Add usage-based charges to invoice |
| `InvoicePaid` | 1 | Handle milestone triggers on payment |
| `OrderCancelled` | 1 | Cancel/skip pending milestones |
| `AdminAreaViewInvoice` | 1 | Display billing info in admin |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement core billing processing
- [ ] Create milestone billing system
- [ ] Add usage-based billing calculations
- [ ] Build rule engine for billing
- [ ] Create admin configuration interface
- [ ] Implement invoice integration
- [ ] Add webhook callbacks for milestones
- [ ] Build reporting functionality
- [ ] Test milestone progression
- [ ] Verify usage calculations
- [ ] Test rule application order
- [ ] Add billing validation logic
- [ ] Create billing audit log
- [ ] Test with various billing scenarios
- [ ] Verify invoice generation
- [ ] Add email notifications for milestones
- [ ] Implement billing export feature
- [ ] Test performance with large datasets