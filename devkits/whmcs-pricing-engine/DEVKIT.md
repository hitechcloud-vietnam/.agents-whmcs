# WHMCS Pricing Engine - Complete Module

## Module Definition

```php
<?php
/**
 * WHMCS Pricing Engine Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function whmcs_pricing_engine_config()
{
    return [
        'name' => 'Pricing Engine',
        'description' => 'Advanced pricing management with dynamic rules and markup',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'default_markup' => [
                'Type' => 'text',
                'Default' => '20',
                'Description' => 'Default markup percentage'
            ],
            'enable_dynamic' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable dynamic pricing'
            ],
            'min_margin' => [
                'Type' => 'text',
                'Default' => '10',
                'Description' => 'Minimum margin percentage'
            ],
            'currency_conversion' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable currency-based pricing'
            ],
            'cost_source' => [
                'Type' => 'dropdown',
                'Default' => 'manual',
                'Options' => [
                    'manual' => 'Manual Entry',
                    'api' => 'API Import',
                    'csv' => 'CSV Import'
                ],
                'Description' => 'Cost data source'
            ]
        ]
    ];
}

function whmcs_pricing_engine_activate()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_pricing_engine_rules` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `rule_name` VARCHAR(255) NOT NULL,
        `rule_type` ENUM('markup','discount','dynamic','time') NOT NULL,
        `conditions` TEXT,
        `markup_percent` DECIMAL(10,2) DEFAULT 0.00,
        `discount_percent` DECIMAL(10,2) DEFAULT 0.00,
        `min_price` DECIMAL(10,2) DEFAULT 0.00,
        `max_price` DECIMAL(10,2) DEFAULT 0.00,
        `priority` INT(11) DEFAULT 0,
        `is_active` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql);

    $sql2 = "CREATE TABLE IF NOT EXISTS `mod_pricing_engine_costs` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `cost` DECIMAL(10,2) NOT NULL,
        `currency` VARCHAR(3) DEFAULT 'USD',
        `effective_date` DATE DEFAULT NULL,
        `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql2);

    $sql3 = "CREATE TABLE IF NOT EXISTS `mod_pricing_engine_log` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) DEFAULT NULL,
        `original_price` DECIMAL(10,2) DEFAULT NULL,
        `calculated_price` DECIMAL(10,2) NOT NULL,
        `rule_applied` VARCHAR(255) DEFAULT NULL,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql3);

    return ['status' => 'success', 'description' => 'Pricing Engine activated'];
}

function whmcs_pricing_engine_deactivate()
{
    return ['status' => 'success', 'description' => 'Pricing Engine deactivated'];
}

function whmcs_pricing_engine_output($vars)
{
    echo '<div class="whmcs-module-admin">';
    echo '<h2>Pricing Engine Configuration</h2>';
    
    // Add new rule form
    echo '<div class="panel panel-default">';
    echo '<div class="panel-heading">Create Pricing Rule</div>';
    echo '<div class="panel-body">';
    echo '<form method="post">';
    echo '<div class="row">';
    echo '<div class="col-md-6">';
    echo '<div class="form-group">';
    echo '<label>Rule Name</label>';
    echo '<input type="text" name="rule_name" class="form-control" required />';
    echo '</div>';
    echo '</div>';
    echo '<div class="col-md-6">';
    echo '<div class="form-group">';
    echo '<label>Rule Type</label>';
    echo '<select name="rule_type" class="form-control">';
    echo '<option value="markup">Markup</option>';
    echo '<option value="discount">Discount</option>';
    echo '<option value="dynamic">Dynamic</option>';
    echo '<option value="time">Time-based</option>';
    echo '</select>';
    echo '</div>';
    echo '</div>';
    echo '</div>';
    
    echo '<div class="row">';
    echo '<div class="col-md-4">';
    echo '<div class="form-group">';
    echo '<label>Markup %</label>';
    echo '<input type="number" step="0.01" name="markup_percent" class="form-control" />';
    echo '</div>';
    echo '</div>';
    echo '<div class="col-md-4">';
    echo '<div class="form-group">';
    echo '<label>Discount %</label>';
    echo '<input type="number" step="0.01" name="discount_percent" class="form-control" />';
    echo '</div>';
    echo '</div>';
    echo '<div class="col-md-4">';
    echo '<div class="form-group">';
    echo '<label>Priority</label>';
    echo '<input type="number" name="priority" class="form-control" value="0" />';
    echo '</div>';
    echo '</div>';
    echo '</div>';
    
    echo '<div class="form-group">';
    echo '<label>Conditions (JSON)</label>';
    echo '<textarea name="conditions" class="form-control" rows="3" placeholder=\'{"min_qty": 10, "currency": "USD"}\'></textarea>';
    echo '</div>';
    
    echo '<button type="submit" name="save_rule" class="btn btn-primary">Save Rule</button>';
    echo '</form>';
    echo '</div>';
    echo '</div>';
    
    // List rules
    echo '<div class="panel panel-default" style="margin-top:20px;">';
    echo '<div class="panel-heading">Active Rules</div>';
    echo '<div class="panel-body">';
    
    $result = select_query("mod_pricing_engine_rules", "*", "", "priority", "DESC");
    if (mysql_num_rows($result) > 0) {
        echo '<table class="datatable">';
        echo '<thead><tr><th>Name</th><th>Type</th><th>Markup</th><th>Discount</th><th>Priority</th><th>Active</th></tr></thead>';
        echo '<tbody>';
        while ($row = mysql_fetch_array($result)) {
            echo '<tr>';
            echo '<td>' . $row['rule_name'] . '</td>';
            echo '<td>' . ucfirst($row['rule_type']) . '</td>';
            echo '<td>' . $row['markup_percent'] . '%</td>';
            echo '<td>' . $row['discount_percent'] . '%</td>';
            echo '<td>' . $row['priority'] . '</td>';
            echo '<td>' . ($row['is_active'] ? 'Yes' : 'No') . '</td>';
            echo '</tr>';
        }
        echo '</tbody></table>';
    } else {
        echo '<p>No pricing rules configured.</p>';
    }
    echo '</div></div>';
    echo '</div>';
}

// Pricing calculation function
function calculateProductPrice($productId, $basePrice, $currency = 'USD')
{
    $markup = get_config('default_markup');
    
    // Get applicable rules sorted by priority
    $rules = select_query("mod_pricing_engine_rules", "*", ["is_active" => 1], "priority", "DESC");
    
    $calculatedPrice = $basePrice;
    $appliedRule = '';
    
    while ($rule = mysql_fetch_array($rules)) {
        if (appliesToProduct($rule, $productId)) {
            if ($rule['rule_type'] === 'markup') {
                $calculatedPrice *= (1 + ($rule['markup_percent'] / 100));
                $appliedRule = $rule['rule_name'];
                break;
            } elseif ($rule['rule_type'] === 'discount') {
                $calculatedPrice *= (1 - ($rule['discount_percent'] / 100));
                $appliedRule = $rule['rule_name'];
                break;
            }
        }
    }
    
    // Apply minimum margin
    $minMargin = get_config('min_margin');
    $minPrice = $basePrice * (1 + ($minMargin / 100));
    if ($calculatedPrice < $minPrice) {
        $calculatedPrice = $minPrice;
    }
    
    // Log calculation
    insert_query("mod_pricing_engine_log", [
        'product_id' => $productId,
        'original_price' => $basePrice,
        'calculated_price' => $calculatedPrice,
        'rule_applied' => $appliedRule
    ]);
    
    return round($calculatedPrice, 2);
}

function appliesToProduct($rule, $productId)
{
    if (empty($rule['conditions'])) {
        return true;
    }
    
    $conditions = json_decode($rule['conditions'], true);
    // Implement condition matching logic here
    return true;
}

function get_config($key)
{
    $result = select_query("mod_pricing_engine_config", "value", ["setting_key" => $key]);
    if ($row = mysql_fetch_array($result)) {
        return $row['value'];
    }
    return null;
}
```

## Hooks

```php
<?php
// Pricing Engine Hooks

add_hook('ProductPriceOverride', 1, function($vars) {
    $pid = $vars['pid'];
    $basePrice = $vars['basePrice'];
    
    return calculateProductPrice($pid, $basePrice, $vars['currency'] ?? 'USD');
});

add_hook('AddToCart', 1, function($vars) {
    $pid = $vars['pid'];
    $result = select_query("mod_pricing_engine_costs", "cost", ["product_id" => $pid], "effective_date", "DESC", "1");
    
    if ($row = mysql_fetch_array($result)) {
        return ['product_cost' => $row['cost']];
    }
});

add_hook('ClientAreaPageProductsOutput', 1, function($vars) {
    return ['pricing_engine_active' => true];
});
```

## Database Schema

```sql
CREATE TABLE `mod_pricing_engine_rules` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `rule_name` VARCHAR(255) NOT NULL,
    `rule_type` ENUM('markup','discount','dynamic','time') NOT NULL,
    `conditions` TEXT,
    `markup_percent` DECIMAL(10,2) DEFAULT 0.00,
    `discount_percent` DECIMAL(10,2) DEFAULT 0.00,
    `min_price` DECIMAL(10,2) DEFAULT 0.00,
    `max_price` DECIMAL(10,2) DEFAULT 0.00,
    `priority` INT(11) DEFAULT 0,
    `is_active` TINYINT(1) DEFAULT 1,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `mod_pricing_engine_costs` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `product_id` INT(11) NOT NULL,
    `cost` DECIMAL(10,2) NOT NULL,
    `currency` VARCHAR(3) DEFAULT 'USD',
    `effective_date` DATE DEFAULT NULL,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `mod_pricing_engine_log` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `product_id` INT(11) DEFAULT NULL,
    `original_price` DECIMAL(10,2) DEFAULT NULL,
    `calculated_price` DECIMAL(10,2) NOT NULL,
    `rule_applied` VARCHAR(255) DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```