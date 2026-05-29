# WHMCS Coupon System - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_coupon_system_config() {
    return [
        'name' => 'Coupon System',
        'description' => 'Full-featured coupon management with code generation',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_validate' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-validate coupons'],
            'show_discount' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show discount preview']
        ]
    ];
}

function whmcs_coupon_system_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_coupon_codes` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `code` VARCHAR(50) NOT NULL UNIQUE,
        `discount_type` ENUM('percentage','fixed') DEFAULT 'percentage',
        `discount_value` DECIMAL(10,2) NOT NULL,
        `min_order` DECIMAL(10,2) DEFAULT 0,
        `max_uses` INT(11) DEFAULT 1,
        `times_used` INT(11) DEFAULT 0,
        `max_per_user` INT(11) DEFAULT 1,
        `start_date` DATE DEFAULT NULL,
        `end_date` DATE DEFAULT NULL,
        `applies_to` ENUM('all','product','category') DEFAULT 'all',
        `applies_id` INT(11) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `code` (`code`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_coupon_usage` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `coupon_id` INT(11) NOT NULL,
        `user_id` INT(11) DEFAULT NULL,
        `order_id` INT(11) DEFAULT NULL,
        `used_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success', 'description' => 'Coupon System activated'];
}

function whmcs_coupon_system_deactivate() {
    return ['status' => 'success'];
}

function whmcs_coupon_system_output($vars) {
    echo '<div class="whmcs-module-admin">
          <h2>Coupon System Management</h2>';
    
    if ($_POST['create_coupon']) {
        $code = strtoupper($_POST['code']) ?: strtoupper(substr(md5(time()), 0, 8));
        insert_query('mod_coupon_codes', [
            'code' => $code,
            'discount_type' => $_POST['discount_type'],
            'discount_value' => $_POST['discount_value'],
            'min_order' => $_POST['min_order'] ?: 0,
            'max_uses' => $_POST['max_uses'] ?: 1,
            'max_per_user' => $_POST['max_per_user'] ?: 1,
            'start_date' => $_POST['start_date'] ?: null,
            'end_date' => $_POST['end_date'] ?: null,
            'applies_to' => $_POST['applies_to'],
            'applies_id' => $_POST['applies_id'] ?: null
        ]);
        echo '<div class="alert alert-success">Coupon created: ' . $code . '</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Coupon</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Code (leave blank for auto)</label>
          <input type="text" name="code" class="form-control" /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Discount Type</label>
          <select name="discount_type" class="form-control">
          <option value="percentage">Percentage</option>
          <option value="fixed">Fixed Amount</option>
          </select></div></div>
          <div class="col-md-4"><div class="form-group"><label>Value</label>
          <input type="number" step="0.01" name="discount_value" class="form-control" required /></div></div>
          </div>
          <div class="row">
          <div class="col-md-3"><div class="form-group"><label>Min Order</label>
          <input type="number" step="0.01" name="min_order" class="form-control" /></div></div>
          <div class="col-md-3"><div class="form-group"><label>Max Uses</label>
          <input type="number" name="max_uses" class="form-control" value="1" /></div></div>
          <div class="col-md-3"><div class="form-group"><label>Max Per User</label>
          <input type="number" name="max_per_user" class="form-control" value="1" /></div></div>
          <div class="col-md-3"><div class="form-group"><label>Applies To</label>
          <select name="applies_to" class="form-control">
          <option value="all">All Products</option>
          <option value="product">Product</option>
          <option value="category">Category</option>
          </select></div></div>
          </div>
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Start Date</label>
          <input type="date" name="start_date" class="form-control" /></div></div>
          <div class="col-md-6"><div class="form-group"><label>End Date</label>
          <input type="date" name="end_date" class="form-control" /></div></div>
          </div>
          <button type="submit" name="create_coupon" class="btn btn-primary">Create Coupon</button>
          </div></form>';
    
    $result = select_query("mod_coupon_codes", "*", "", "id", "DESC");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Code</th><th>Discount</th><th>Used</th><th>Valid</th><th>Status</th></tr></thead>
          <tbody>';
    while ($row = mysql_fetch_array($result)) {
        $now = date('Y-m-d');
        $valid = (!$row['start_date'] || $row['start_date'] <= $now) && (!$row['end_date'] || $row['end_date'] >= $now);
        $validity = $valid ? 'Valid' : 'Expired';
        echo '<tr><td><strong>' . $row['code'] . '</strong></td>
              <td>' . ($row['discount_type'] == 'percentage' ? $row['discount_value'] . '%' : '$' . $row['discount_value']) . '</td>
              <td>' . $row['times_used'] . '/' . $row['max_uses'] . '</td>
              <td>' . $validity . '</td>
              <td>' . ($row['is_active'] ? 'Active' : 'Inactive') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ValidateCouponCode', 1, function($vars) {
    $code = strtoupper($vars['code']);
    $result = select_query("mod_coupon_codes", "*", ["code" => $code, "is_active" => 1]);
    if ($coupon = mysql_fetch_array($result)) {
        $now = date('Y-m-d');
        if ($coupon['start_date'] && $coupon['start_date'] > $now) return ['valid' => false, 'error' => 'Coupon not yet valid'];
        if ($coupon['end_date'] && $coupon['end_date'] < $now) return ['valid' => false, 'error' => 'Coupon expired'];
        if ($coupon['max_uses'] && $coupon['times_used'] >= $coupon['max_uses']) return ['valid' => false, 'error' => 'Coupon usage limit reached'];
        return ['valid' => true, 'discount' => $coupon['discount_value'], 'type' => $coupon['discount_type']];
    }
    return ['valid' => false, 'error' => 'Invalid coupon code'];
});

add_hook('ApplyCoupon', 1, function($vars) {
    $code = strtoupper($vars['code']);
    update_query("mod_coupon_codes", ["times_used" => "+=1"], ["code" => $code]);
    insert_query("mod_coupon_usage", ["coupon_id" => $vars['coupon_id'], "user_id" => $_SESSION['uid'], "order_id" => $vars['order_id']]);
});
```

## Database

```sql
CREATE TABLE `mod_coupon_codes` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `code` VARCHAR(50) NOT NULL UNIQUE,
    `discount_type` ENUM('percentage','fixed') DEFAULT 'percentage',
    `discount_value` DECIMAL(10,2) NOT NULL,
    `min_order` DECIMAL(10,2) DEFAULT 0,
    `max_uses` INT(11) DEFAULT 1,
    `times_used` INT(11) DEFAULT 0,
    `max_per_user` INT(11) DEFAULT 1,
    `start_date` DATE DEFAULT NULL,
    `end_date` DATE DEFAULT NULL,
    `applies_to` ENUM('all','product','category') DEFAULT 'all',
    `applies_id` INT(11) DEFAULT NULL,
    `is_active` TINYINT(1) DEFAULT 1,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```