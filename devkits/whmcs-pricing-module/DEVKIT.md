# WHMCS Pricing Module DevKit

## Header

**Purpose:** Dynamic pricing engine that calculates product/service prices based on configurable rules, quantity tiers, customer groups, and time-based promotions.

**Module Type:** Pricing/Configuration Module

**Use Case:** Hosting companies that offer volume discounts, tiered pricing, promotional rates, or need complex pricing calculations beyond WHMCS default pricing.

---

## Complete Code Template

### File Structure
```
whmcs-pricing-module/
├── README.md
├── DEVKIT.md
├── pricing.php              # Main pricing logic
├── hooks.php               # WHMCS hook integrations
└── templates/
    └── admin_pricing.tpl   # Admin configuration template
```

### Main Module File: pricing.php

```php
<?php
/**
 * WHMCS Dynamic Pricing Module
 * 
 * Provides configurable pricing rules engine for products and services.
 * Supports quantity tiers, customer groups, and time-based pricing.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('PRICING_MODULE_VERSION', '1.0.0');

// Pricing Rule Types
define('PRICING_RULE_FIXED', 'fixed');
define('PRICING_RULE_PERCENTAGE', 'percentage');
define('PRICING_RULE_TIERED', 'tiered');
define('PRICING_RULE_VOLUME', 'volume');

// Pricing Rule Status
define('PRICING_STATUS_ACTIVE', 'active');
define('PRICING_STATUS_INACTIVE', 'inactive');
define('PRICING_STATUS_SCHEDULED', 'scheduled');

/**
 * Calculate price with all active pricing rules applied
 * 
 * @param int $productId Product ID
 * @param int $quantity Quantity being purchased
 * @param int $clientId Client ID (0 for anonymous)
 * @param string $cycle Billing cycle (monthly, quarterly, annually, etc.)
 * @param array $additionalData Additional context data
 * @return float Final calculated price
 */
function pricing_calculate($productId, $quantity = 1, $clientId = 0, $cycle = 'monthly', $additionalData = []) {
    // Get base price from WHMCS
    $basePrice = pricing_get_base_price($productId, $cycle);
    
    // Get client group discount if applicable
    $clientGroupDiscount = pricing_get_client_group_discount($clientId);
    
    // Get quantity tier discount
    $quantityDiscount = pricing_get_quantity_tier_discount($productId, $quantity);
    
    // Get promotional pricing
    $promoDiscount = pricing_get_promotional_discount($productId, $cycle);
    
    // Get time-based pricing
    $timeDiscount = pricing_get_time_based_discount($productId, $cycle);
    
    // Apply all rules in priority order
    $finalPrice = $basePrice;
    
    // Apply client group discount first
    if ($clientGroupDiscount > 0) {
        $finalPrice = pricing_apply_discount($finalPrice, $clientGroupDiscount, PRICING_RULE_PERCENTAGE);
    }
    
    // Apply quantity tier discount
    if ($quantityDiscount > 0) {
        $finalPrice = pricing_apply_discount($finalPrice, $quantityDiscount, PRICING_RULE_PERCENTAGE);
    }
    
    // Apply promotional discount
    if ($promoDiscount['type'] == PRICING_RULE_PERCENTAGE) {
        $finalPrice = pricing_apply_discount($finalPrice, $promoDiscount['value'], PRICING_RULE_PERCENTAGE);
    } elseif ($promoDiscount['type'] == PRICING_RULE_FIXED) {
        $finalPrice = $promoDiscount['value'];
    }
    
    // Apply time-based discount
    if ($timeDiscount['type'] == PRICING_RULE_PERCENTAGE) {
        $finalPrice = pricing_apply_discount($finalPrice, $timeDiscount['value'], PRICING_RULE_PERCENTAGE);
    }
    
    // Apply minimum price floor
    $minPrice = pricing_get_minimum_price($productId);
    $finalPrice = max($finalPrice, $minPrice);
    
    return round($finalPrice, 2);
}

/**
 * Get base price from WHMCS pricing table
 */
function pricing_get_base_price($productId, $cycle) {
    $query = "SELECT `price` FROM `tblpricing` WHERE `relid` = ? AND `type` = 'product' AND `tsetupfee` = 0";
    $result = full_query($query);
    $data = mysql_fetch_assoc($result);
    return $data['price'] ?? 0;
}

/**
 * Get client group discount percentage
 */
function pricing_get_client_group_discount($clientId) {
    if ($clientId == 0) {
        return 0;
    }
    
    $query = "SELECT `cg`.`discount` FROM `tblclients` `c`
              JOIN `tblclientgroups` `cg` ON `c`.`groupid` = `cg`.`id`
              WHERE `c`.`id` = ?";
    $result = full_query($query, [$clientId]);
    $data = mysql_fetch_assoc($result);
    return $data['discount'] ?? 0;
}

/**
 * Get quantity-based tier discount
 */
function pricing_get_quantity_tier_discount($productId, $quantity) {
    $query = "SELECT `discount_percent` FROM `mod_pricing_tiers`
              WHERE `product_id` = ? AND `min_quantity` <= ? AND `max_quantity` >= ?
              AND `status` = 'active'
              ORDER BY `min_quantity` DESC LIMIT 1";
    $result = full_query($query, [$productId, $quantity, $quantity]);
    $data = mysql_fetch_assoc($result);
    return $data['discount_percent'] ?? 0;
}

/**
 * Get active promotional discount
 */
function pricing_get_promotional_discount($productId, $cycle) {
    $query = "SELECT `discount_type`, `discount_value` FROM `mod_pricing_promotions`
              WHERE `product_id` = ? AND `billing_cycle` = ?
              AND `start_date` <= NOW() AND `end_date` >= NOW()
              AND `status` = 'active'
              ORDER BY `priority` DESC LIMIT 1";
    $result = full_query($query, [$productId, $cycle]);
    $data = mysql_fetch_assoc($result);
    
    if ($data) {
        return [
            'type' => $data['discount_type'],
            'value' => $data['discount_value']
        ];
    }
    
    return ['type' => null, 'value' => 0];
}

/**
 * Get time-based pricing (early bird, last minute, etc.)
 */
function pricing_get_time_based_discount($productId, $cycle) {
    $hour = date('H');
    
    // Early bird discount (before 9 AM)
    if ($hour < 9) {
        $query = "SELECT `discount_type`, `discount_value` FROM `mod_pricing_time_rules`
                  WHERE `rule_type` = 'early_bird' AND `status` = 'active' LIMIT 1";
        $result = full_query($query);
        $data = mysql_fetch_assoc($result);
        if ($data) {
            return ['type' => $data['discount_type'], 'value' => $data['discount_value']];
        }
    }
    
    // Last minute discount (after 6 PM)
    if ($hour >= 18) {
        $query = "SELECT `discount_type`, `discount_value` FROM `mod_pricing_time_rules`
                  WHERE `rule_type` = 'last_minute' AND `status` = 'active' LIMIT 1";
        $result = full_query($query);
        $data = mysql_fetch_assoc($result);
        if ($data) {
            return ['type' => $data['discount_type'], 'value' => $data['discount_value']];
        }
    }
    
    return ['type' => null, 'value' => 0];
}

/**
 * Apply discount to price
 */
function pricing_apply_discount($price, $discount, $type) {
    if ($type == PRICING_RULE_PERCENTAGE) {
        return $price * (1 - ($discount / 100));
    } elseif ($type == PRICING_RULE_FIXED) {
        return $price - $discount;
    }
    return $price;
}

/**
 * Get minimum price floor for product
 */
function pricing_get_minimum_price($productId) {
    $query = "SELECT `minimum_price` FROM `mod_pricing_floors` WHERE `product_id` = ?";
    $result = full_query($query, [$productId]);
    $data = mysql_fetch_assoc($result);
    return $data['minimum_price'] ?? 0;
}

/**
 * Create a new pricing rule
 */
function pricing_create_rule($data) {
    $fields = [
        'product_id', 'rule_type', 'discount_type', 'discount_value',
        'min_quantity', 'max_quantity', 'client_group_id', 'billing_cycle',
        'start_date', 'end_date', 'priority', 'status', 'created_at'
    ];
    
    $values = [
        $data['product_id'], $data['rule_type'], $data['discount_type'],
        $data['discount_value'], $data['min_quantity'] ?? 0, $data['max_quantity'] ?? 999,
        $data['client_group_id'] ?? 0, $data['billing_cycle'] ?? 'monthly',
        $data['start_date'] ?? date('Y-m-d'), $data['end_date'] ?? null,
        $data['priority'] ?? 50, PRICING_STATUS_ACTIVE, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_pricing_rules', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Update existing pricing rule
 */
function pricing_update_rule($ruleId, $data) {
    $updateData = [];
    
    $allowedFields = ['rule_type', 'discount_type', 'discount_value', 
                     'min_quantity', 'max_quantity', 'status', 'end_date'];
    
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            $updateData[$field] = $data[$field];
        }
    }
    
    $updateData['updated_at'] = date('Y-m-d H:i:s');
    
    update_query('mod_pricing_rules', $updateData, ['id' => $ruleId]);
}

/**
 * Delete pricing rule
 */
function pricing_delete_rule($ruleId) {
    delete_query('mod_pricing_rules', ['id' => $ruleId]);
}

/**
 * Get all pricing rules for admin display
 */
function pricing_get_all_rules($filters = []) {
    $where = "1=1";
    $params = [];
    
    if (!empty($filters['product_id'])) {
        $where .= " AND pr.product_id = ?";
        $params[] = $filters['product_id'];
    }
    
    if (!empty($filters['status'])) {
        $where .= " AND pr.status = ?";
        $params[] = $filters['status'];
    }
    
    $query = "SELECT pr.*, p.name as product_name, cg.groupname 
              FROM mod_pricing_rules pr
              LEFT JOIN tblproducts p ON pr.product_id = p.id
              LEFT JOIN tblclientgroups cg ON pr.client_group_id = cg.id
              WHERE {$where}
              ORDER BY pr.priority DESC, pr.created_at DESC";
    
    $result = full_query($query, $params);
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = $row;
    }
    
    return $rules;
}

/**
 * Get pricing rule by ID
 */
function pricing_get_rule($ruleId) {
    $query = "SELECT * FROM mod_pricing_rules WHERE id = ?";
    $result = full_query($query, [$ruleId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get pricing history for audit
 */
function pricing_get_history($productId, $limit = 100) {
    $query = "SELECT * FROM mod_pricing_history 
              WHERE product_id = ? 
              ORDER BY created_at DESC LIMIT ?";
    $result = full_query($query, [$productId, $limit]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $history[] = $row;
    }
    
    return $history;
}

/**
 * Log pricing calculation for analytics
 */
function pricing_log_calculation($data) {
    $fields = ['product_id', 'client_id', 'quantity', 'base_price', 
               'final_price', 'applied_rules', 'cycle', 'created_at'];
    
    $values = [
        $data['product_id'], $data['client_id'] ?? 0, $data['quantity'],
        $data['base_price'], $data['final_price'], json_encode($data['applied_rules']),
        $data['cycle'], date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_pricing_calculations', array_combine($fields, $values));
}

/**
 * Get pricing analytics summary
 */
function pricing_get_analytics($startDate, $endDate) {
    $query = "SELECT 
                COUNT(*) as total_calculations,
                AVG(final_price) as avg_price,
                SUM(base_price - final_price) as total_discount_given,
                product_id,
                p.name as product_name
              FROM mod_pricing_calculations pc
              LEFT JOIN tblproducts p ON pc.product_id = p.id
              WHERE pc.created_at BETWEEN ? AND ?
              GROUP BY product_id
              ORDER BY total_calculations DESC";
    
    $result = full_query($query, [$startDate, $endDate]);
    
    $analytics = [];
    while ($row = mysql_fetch_assoc($result)) {
        $analytics[] = $row;
    }
    
    return $analytics;
}

/**
 * Validate pricing rule configuration
 */
function pricing_validate_rule($data, &$errors) {
    if (empty($data['product_id'])) {
        $errors[] = "Product ID is required";
    }
    
    if (empty($data['discount_value']) && $data['discount_value'] !== 0) {
        $errors[] = "Discount value is required";
    }
    
    if ($data['discount_type'] == PRICING_RULE_PERCENTAGE && $data['discount_value'] > 100) {
        $errors[] = "Percentage discount cannot exceed 100%";
    }
    
    if (!empty($data['start_date']) && !empty($data['end_date'])) {
        if (strtotime($data['end_date']) < strtotime($data['start_date'])) {
            $errors[] = "End date must be after start date";
        }
    }
    
    return count($errors) === 0;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Pricing Module Hooks
 */

// Hook: Calculate product price during cart
add_hook('ShoppingCartValidateProduct', 1, function($params) {
    $productId = $params['productId'];
    $quantity = $params['qty'];
    $clientId = $_SESSION['uid'] ?? 0;
    
    $finalPrice = pricing_calculate($productId, $quantity, $clientId, $params['billingcycle']);
    
    // Override the price in the cart
    return [
        'price' => $finalPrice
    ];
});

// Hook: Apply pricing after order is placed
add_hook('OrderProductPricingOverride', 1, function($params) {
    $productId = $params['productId'];
    $clientId = $params['clientId'];
    $quantity = $params['qty'];
    
    $finalPrice = pricing_calculate($productId, $quantity, $clientId, $params['billingcycle']);
    
    return [
        'override' => true,
        'price' => $finalPrice
    ];
});

// Hook: Log pricing when invoice created
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Fetch invoice items and log pricing
    $query = "SELECT inv.*, invi.* FROM tblinvoices inv
              JOIN tblinvoiceitems invi ON inv.id = invi.invoice
              WHERE inv.id = ?";
    $result = full_query($query, [$invoiceId]);
    
    while ($item = mysql_fetch_assoc($result)) {
        if ($item['type'] == 'Hosting' || $item['type'] == 'Product') {
            pricing_log_calculation([
                'product_id' => $item['relid'],
                'client_id' => $item['userid'],
                'quantity' => 1,
                'base_price' => $item['amount'],
                'final_price' => $item['amount'],
                'applied_rules' => ['invoice_created' => true],
                'cycle' => $item['billingcycle'] ?? 'monthly'
            ]);
        }
    }
});

// Hook: Apply pricing to quote
add_hook('QuoteValidateItem', 1, function($params) {
    $productId = $params['productId'];
    $clientId = $params['clientId'];
    
    $finalPrice = pricing_calculate($productId, 1, $clientId, 'monthly');
    
    return ['price' => $finalPrice];
});

// Hook: Admin area pricing modification
add_hook('AdminAreaViewPricingOverride', 1, function($params) {
    if (!empty($params['productId'])) {
        return pricing_calculate(
            $params['productId'],
            $params['quantity'] ?? 1,
            $params['clientId'] ?? 0,
            $params['cycle'] ?? 'monthly'
        );
    }
});
```

### Admin Template: templates/admin_pricing.tpl

```html
{extends file="admin/template.tpl"}

{block name="content"}
<div class="pricing-admin">
    <div class="header">
        <h2>Dynamic Pricing Configuration</h2>
        <button class="btn btn-primary" onclick="showAddRuleModal()">Add Pricing Rule</button>
    </div>

    <div class="pricing-filters">
        <select id="filter-product">
            <option value="">All Products</option>
            {foreach from=$products item=p}
            <option value="{$p.id}">{$p.name}</option>
            {/foreach}
        </select>
        <select id="filter-status">
            <option value="">All Status</option>
            <option value="active">Active</option>
            <option value="inactive">Inactive</option>
            <option value="scheduled">Scheduled</option>
        </select>
        <button class="btn" onclick="filterRules()">Filter</button>
    </div>

    <table class="pricing-rules-table">
        <thead>
            <tr>
                <th>Product</th>
                <th>Rule Type</th>
                <th>Discount</th>
                <th>Quantity Range</th>
                <th>Client Group</th>
                <th>Valid Period</th>
                <th>Status</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach from=$rules item=rule}
            <tr data-rule-id="{$rule.id}">
                <td>{$rule.product_name}</td>
                <td>{$rule.rule_type}</td>
                <td>
                    {if $rule.discount_type == 'percentage'}
                        {$rule.discount_value}%
                    {else}
                        {$currency_prefix}{$rule.discount_value}{$currency_suffix}
                    {/if}
                </td>
                <td>{$rule.min_quantity} - {$rule.max_quantity}</td>
                <td>{$rule.groupname|default:'All Groups'}</td>
                <td>{$rule.start_date} to {$rule.end_date}</td>
                <td>
                    <span class="status-badge status-{$rule.status}">{$rule.status}</span>
                </td>
                <td>
                    <button onclick="editRule({$rule.id})">Edit</button>
                    <button onclick="deleteRule({$rule.id})">Delete</button>
                    <button onclick="toggleStatus({$rule.id})">Toggle</button>
                </td>
            </tr>
            {/foreach}
        </tbody>
    </table>

    <div class="analytics-section">
        <h3>Pricing Analytics</h3>
        <div class="analytics-grid">
            <div class="analytics-card">
                <h4>Total Calculations</h4>
                <p class="analytics-value">{$analytics.total_calculations}</p>
            </div>
            <div class="analytics-card">
                <h4>Average Price</h4>
                <p class="analytics-value">{$currency_prefix}{$analytics.avg_price}{$currency_suffix}</p>
            </div>
            <div class="analytics-card">
                <h4>Total Discount Given</h4>
                <p class="analytics-value">{$currency_prefix}{$analytics.total_discount_given}{$currency_suffix}</p>
            </div>
        </div>
    </div>
</div>

<!-- Add/Edit Rule Modal -->
<div id="rule-modal" class="modal" style="display:none;">
    <div class="modal-content">
        <h3>{if $edit_id}Edit{else}Add{/if} Pricing Rule</h3>
        <form id="rule-form">
            <input type="hidden" name="id" value="{$edit_id}">
            
            <div class="form-group">
                <label>Product</label>
                <select name="product_id" required>
                    <option value="">Select Product</option>
                    {foreach from=$products item=p}
                    <option value="{$p.id}">{$p.name}</option>
                    {/foreach}
                </select>
            </div>

            <div class="form-group">
                <label>Rule Type</label>
                <select name="rule_type" required>
                    <option value="quantity_tier">Quantity Tier</option>
                    <option value="client_group">Client Group</option>
                    <option value="promotional">Promotional</option>
                    <option value="time_based">Time Based</option>
                </select>
            </div>

            <div class="form-group">
                <label>Discount Type</label>
                <select name="discount_type" required>
                    <option value="percentage">Percentage</option>
                    <option value="fixed">Fixed Amount</option>
                </select>
            </div>

            <div class="form-group">
                <label>Discount Value</label>
                <input type="number" name="discount_value" step="0.01" required>
            </div>

            <div class="form-row">
                <div class="form-group">
                    <label>Min Quantity</label>
                    <input type="number" name="min_quantity" value="1">
                </div>
                <div class="form-group">
                    <label>Max Quantity</label>
                    <input type="number" name="max_quantity" value="999">
                </div>
            </div>

            <div class="form-group">
                <label>Client Group</label>
                <select name="client_group_id">
                    <option value="">All Groups</option>
                    {foreach from=$clientGroups item=cg}
                    <option value="{$cg.id}">{$cg.groupname}</option>
                    {/foreach}
                </select>
            </div>

            <div class="form-row">
                <div class="form-group">
                    <label>Start Date</label>
                    <input type="date" name="start_date">
                </div>
                <div class="form-group">
                    <label>End Date</label>
                    <input type="date" name="end_date">
                </div>
            </div>

            <div class="form-group">
                <label>Priority (1-100)</label>
                <input type="number" name="priority" value="50" min="1" max="100">
            </div>

            <div class="form-actions">
                <button type="submit" class="btn btn-primary">Save Rule</button>
                <button type="button" class="btn" onclick="closeModal()">Cancel</button>
            </div>
        </form>
    </div>
</div>
{/block}
```

---

## Database Schema

### Tables for Pricing Module

```sql
-- Main pricing rules table
CREATE TABLE `mod_pricing_rules` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL,
    `rule_type` ENUM('quantity_tier', 'client_group', 'promotional', 'time_based') NOT NULL,
    `discount_type` ENUM('percentage', 'fixed') NOT NULL DEFAULT 'percentage',
    `discount_value` DECIMAL(10,2) NOT NULL,
    `min_quantity` INT DEFAULT 1,
    `max_quantity` INT DEFAULT 999,
    `client_group_id` INT DEFAULT NULL,
    `billing_cycle` VARCHAR(20) DEFAULT 'monthly',
    `start_date` DATE DEFAULT NULL,
    `end_date` DATE DEFAULT NULL,
    `priority` INT DEFAULT 50,
    `status` ENUM('active', 'inactive', 'scheduled') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_product_status` (`product_id`, `status`),
    INDEX `idx_rule_type` (`rule_type`),
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE CASCADE,
    FOREIGN KEY (`client_group_id`) REFERENCES `tblclientgroups`(`id`) ON DELETE SET NULL
);

-- Quantity pricing tiers
CREATE TABLE `mod_pricing_tiers` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL,
    `min_quantity` INT NOT NULL,
    `max_quantity` INT NOT NULL,
    `discount_percent` DECIMAL(5,2) NOT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE CASCADE
);

-- Promotional pricing
CREATE TABLE `mod_pricing_promotions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL,
    `promotion_name` VARCHAR(255) NOT NULL,
    `discount_type` ENUM('percentage', 'fixed') NOT NULL,
    `discount_value` DECIMAL(10,2) NOT NULL,
    `billing_cycle` VARCHAR(20) DEFAULT 'monthly',
    `start_date` DATETIME NOT NULL,
    `end_date` DATETIME NOT NULL,
    `priority` INT DEFAULT 50,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE CASCADE
);

-- Time-based pricing rules
CREATE TABLE `mod_pricing_time_rules` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `rule_type` VARCHAR(50) NOT NULL,
    `discount_type` ENUM('percentage', 'fixed') NOT NULL,
    `discount_value` DECIMAL(10,2) NOT NULL,
    `days_of_week` VARCHAR(50) DEFAULT NULL,
    `start_hour` INT DEFAULT 0,
    `end_hour` INT DEFAULT 23,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Minimum price floors
CREATE TABLE `mod_pricing_floors` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL UNIQUE,
    `minimum_price` DECIMAL(10,2) NOT NULL,
    `currency_id` INT DEFAULT 1,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE CASCADE
);

-- Pricing calculation log
CREATE TABLE `mod_pricing_calculations` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL,
    `client_id` INT DEFAULT NULL,
    `quantity` INT DEFAULT 1,
    `base_price` DECIMAL(10,2) NOT NULL,
    `final_price` DECIMAL(10,2) NOT NULL,
    `applied_rules` JSON DEFAULT NULL,
    `cycle` VARCHAR(20) DEFAULT 'monthly',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_product_date` (`product_id`, `created_at`),
    INDEX `idx_client_date` (`client_id`, `created_at`)
);

-- Pricing history for audit
CREATE TABLE `mod_pricing_history` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `product_id` INT NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `old_value` DECIMAL(10,2) DEFAULT NULL,
    `new_value` DECIMAL(10,2) DEFAULT NULL,
    `changed_by` INT DEFAULT NULL,
    `reason` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`product_id`) REFERENCES `tblproducts`(`id`) ON DELETE CASCADE
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ShoppingCartValidateProduct` | 1 | Override cart product pricing |
| `OrderProductPricingOverride` | 1 | Apply pricing at order placement |
| `InvoiceCreationPreCheck` | 1 | Log pricing when invoice created |
| `QuoteValidateItem` | 1 | Apply pricing to quotes |
| `AdminAreaViewPricingOverride` | 1 | Admin area price modifications |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement core pricing calculation function
- [ ] Create admin configuration page
- [ ] Add quantity tier pricing support
- [ ] Add client group discount integration
- [ ] Implement promotional pricing
- [ ] Add time-based pricing rules
- [ ] Create pricing history logging
- [ ] Implement minimum price floor
- [ ] Add analytics dashboard
- [ ] Test pricing with multiple rules
- [ ] Verify hook integrations
- [ ] Test edge cases (0 discount, negative values)
- [ ] Add pricing rule validation
- [ ] Create pricing export functionality
- [ ] Test with different currencies
- [ ] Verify performance with large rule sets
- [ ] Add caching for frequent calculations
- [ ] Document configuration options