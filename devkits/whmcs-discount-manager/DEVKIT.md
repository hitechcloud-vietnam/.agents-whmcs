# WHMCS Discount Manager - Complete Module

## Module Definition

```php
<?php
/**
 * WHMCS Discount Manager Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function whmcs_discount_manager_config()
{
    return [
        'name' => 'Discount Manager',
        'description' => 'Comprehensive discount management with rules and promotions',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_apply' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Automatically apply discounts'
            ],
            'stack_discounts' => [
                'Type' => 'yesno',
                'Default' => 'off',
                'Description' => 'Allow discount stacking'
            ],
            'show_original' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Show original price with discount'
            ],
            'max_discount' => [
                'Type' => 'text',
                'Default' => '50',
                'Description' => 'Maximum discount percentage allowed'
            ]
        ]
    ];
}

function whmcs_discount_manager_activate()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_discount_manager_rules` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `rule_name` VARCHAR(255) NOT NULL,
        `discount_type` ENUM('percentage','fixed','bogo') NOT NULL,
        `discount_value` DECIMAL(10,2) NOT NULL,
        `min_purchase` DECIMAL(10,2) DEFAULT 0.00,
        `max_discount` DECIMAL(10,2) DEFAULT NULL,
        `applies_to` ENUM('all','product','category','customer') DEFAULT 'all',
        `applies_id` INT(11) DEFAULT NULL,
        `customer_group_id` INT(11) DEFAULT NULL,
        `start_date` DATETIME DEFAULT NULL,
        `end_date` DATETIME DEFAULT NULL,
        `usage_limit` INT(11) DEFAULT NULL,
        `times_used` INT(11) DEFAULT 0,
        `is_active` TINYINT(1) DEFAULT 1,
        `priority` INT(11) DEFAULT 0,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql);

    $sql2 = "CREATE TABLE IF NOT EXISTS `mod_discount_manager_usage` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `rule_id` INT(11) NOT NULL,
        `user_id` INT(11) DEFAULT NULL,
        `order_id` INT(11) DEFAULT NULL,
        `discount_amount` DECIMAL(10,2) NOT NULL,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `rule_id` (`rule_id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql2);

    return ['status' => 'success', 'description' => 'Discount Manager activated'];
}

function whmcs_discount_manager_deactivate()
{
    return ['status' => 'success'];
}

// Admin output
function whmcs_discount_manager_output($vars)
{
    echo '<div class="whmcs-module-admin">';
    echo '<h2>Discount Manager</h2>';
    
    // Create discount form
    echo '<div class="panel panel-default">';
    echo '<div class="panel-heading">Create New Discount Rule</div>';
    echo '<div class="panel-body">';
    echo '<form method="post">';
    echo '<div class="row">';
    echo '<div class="col-md-6">';
    echo '<div class="form-group"><label>Rule Name</label>';
    echo '<input type="text" name="rule_name" class="form-control" required /></div>';
    echo '</div>';
    echo '<div class="col-md-6">';
    echo '<div class="form-group"><label>Discount Type</label>';
    echo '<select name="discount_type" class="form-control">';
    echo '<option value="percentage">Percentage</option>';
    echo '<option value="fixed">Fixed Amount</option>';
    echo '<option value="bogo">Buy One Get One</option>';
    echo '</select></div>';
    echo '</div>';
    echo '</div>';
    
    echo '<div class="row">';
    echo '<div class="col-md-4">';
    echo '<div class="form-group"><label>Discount Value</label>';
    echo '<input type="number" step="0.01" name="discount_value" class="form-control" required /></div>';
    echo '</div>';
    echo '<div class="col-md-4">';
    echo '<div class="form-group"><label>Applies To</label>';
    echo '<select name="applies_to" class="form-control">';
    echo '<option value="all">All Products</option>';
    echo '<option value="product">Specific Product</option>';
    echo '<option value="category">Category</option>';
    echo '<option value="customer">Customer Group</option>';
    echo '</select></div>';
    echo '</div>';
    echo '<div class="col-md-4">';
    echo '<div class="form-group"><label>Priority</label>';
    echo '<input type="number" name="priority" class="form-control" value="0" /></div>';
    echo '</div>';
    echo '</div>';
    
    echo '<div class="row">';
    echo '<div class="col-md-6">';
    echo '<div class="form-group"><label>Start Date</label>';
    echo '<input type="datetime-local" name="start_date" class="form-control" /></div>';
    echo '</div>';
    echo '<div class="col-md-6">';
    echo '<div class="form-group"><label>End Date</label>';
    echo '<input type="datetime-local" name="end_date" class="form-control" /></div>';
    echo '</div>';
    echo '</div>';
    
    echo '<button type="submit" name="save_discount" class="btn btn-primary">Save Discount Rule</button>';
    echo '</form>';
    echo '</div></div>';
    
    // List discounts
    $result = select_query("mod_discount_manager_rules", "*", "", "priority", "DESC");
    echo '<table class="datatable" style="margin-top:20px;">';
    echo '<thead><tr><th>Name</th><th>Type</th><th>Value</th><th>Applies To</th><th>Valid Until</th><th>Used</th><th>Active</th></tr></thead>';
    echo '<tbody>';
    while ($row = mysql_fetch_array($result)) {
        echo '<tr>';
        echo '<td>' . htmlspecialchars($row['rule_name']) . '</td>';
        echo '<td>' . ucfirst($row['discount_type']) . '</td>';
        echo '<td>' . ($row['discount_type'] == 'percentage' ? $row['discount_value'] . '%' : '$' . $row['discount_value']) . '</td>';
        echo '<td>' . ucfirst($row['applies_to']) . '</td>';
        echo '<td>' . ($row['end_date'] ?: 'Unlimited') . '</td>';
        echo '<td>' . $row['times_used'] . '</td>';
        echo '<td>' . ($row['is_active'] ? '<span style="color:green">Yes</span>' : '<span style="color:red">No</span>') . '</td>';
        echo '</tr>';
    }
    echo '</tbody></table>';
    echo '</div>';
}

// Calculate discount
function calculateDiscount($originalPrice, $productId = null, $userId = null)
{
    $rules = select_query("mod_discount_manager_rules", "*", 
        ["is_active" => 1], "priority", "DESC");
    
    $discount = 0;
    $appliedRule = null;
    
    while ($rule = mysql_fetch_array($rules)) {
        if (!isValidDiscountRule($rule, $productId, $userId)) {
            continue;
        }
        
        if ($rule['discount_type'] == 'percentage') {
            $discount = $originalPrice * ($rule['discount_value'] / 100);
        } elseif ($rule['discount_type'] == 'fixed') {
            $discount = $rule['discount_value'];
        }
        
        $appliedRule = $rule['rule_name'];
        break;
    }
    
    // Respect max discount limit
    $maxDiscount = get_config('max_discount');
    $maxAllowed = $originalPrice * ($maxDiscount / 100);
    if ($discount > $maxAllowed) {
        $discount = $maxAllowed;
    }
    
    return [
        'original_price' => $originalPrice,
        'discount_amount' => round($discount, 2),
        'final_price' => round($originalPrice - $discount, 2),
        'applied_rule' => $appliedRule
    ];
}

function isValidDiscountRule($rule, $productId, $userId)
{
    $now = date('Y-m-d H:i:s');
    
    // Check date range
    if ($rule['start_date'] && $now < $rule['start_date']) return false;
    if ($rule['end_date'] && $now > $rule['end_date']) return false;
    
    // Check usage limit
    if ($rule['usage_limit'] && $rule['times_used'] >= $rule['usage_limit']) return false;
    
    return true;
}
```

## Hooks

```php
<?php
add_hook('OrderFormViewCart', 1, function($vars) {
    return ['discount_manager_active' => true];
});

add_hook('CalculateItemPrice', 1, function($vars) {
    $price = $vars['price'];
    $pid = $vars['product_id'] ?? null;
    $uid = $_SESSION['uid'] ?? null;
    
    return calculateDiscount($price, $pid, $uid);
});

add_hook('InvoiceCreation', 1, function($vars) {
    // Apply discounts to invoice items
});
```

## Database Schema

```sql
CREATE TABLE `mod_discount_manager_rules` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `rule_name` VARCHAR(255) NOT NULL,
    `discount_type` ENUM('percentage','fixed','bogo') NOT NULL,
    `discount_value` DECIMAL(10,2) NOT NULL,
    `min_purchase` DECIMAL(10,2) DEFAULT 0.00,
    `max_discount` DECIMAL(10,2) DEFAULT NULL,
    `applies_to` ENUM('all','product','category','customer') DEFAULT 'all',
    `applies_id` INT(11) DEFAULT NULL,
    `customer_group_id` INT(11) DEFAULT NULL,
    `start_date` DATETIME DEFAULT NULL,
    `end_date` DATETIME DEFAULT NULL,
    `usage_limit` INT(11) DEFAULT NULL,
    `times_used` INT(11) DEFAULT 0,
    `is_active` TINYINT(1) DEFAULT 1,
    `priority` INT(11) DEFAULT 0,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `mod_discount_manager_usage` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `rule_id` INT(11) NOT NULL,
    `user_id` INT(11) DEFAULT NULL,
    `order_id` INT(11) DEFAULT NULL,
    `discount_amount` DECIMAL(10,2) NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `rule_id` (`rule_id`),
    KEY `user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```