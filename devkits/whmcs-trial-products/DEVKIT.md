# WHMCS Trial Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_trial_products_config() {
    return [
        'name' => 'Trial Products',
        'description' => 'Manage trial products with automatic conversion',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_convert' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-convert after trial'],
            'reminder_days' => ['Type' => 'text', 'Default' => '3', 'Description' => 'Days before trial ends to remind']
        ]
    ];
}

function whmcs_trial_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_trial_products` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `trial_days` INT(11) NOT NULL DEFAULT 14,
        `trial_price` DECIMAL(10,2) DEFAULT 0.00,
        `convert_to_product_id` INT(11) DEFAULT NULL,
        `convert_price` DECIMAL(10,2) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_trial_instances` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `service_id` INT(11) NOT NULL,
        `trial_start` DATE NOT NULL,
        `trial_end` DATE NOT NULL,
        `status` ENUM('active','expired','converted','cancelled') DEFAULT 'active',
        `reminder_sent` TINYINT(1) DEFAULT 0,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `service_id` (`service_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_trial_products_deactivate() { return ['status' => 'success']; }

function whmcs_trial_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Trial Products</h2>';
    
    if ($_POST['configure_trial']) {
        insert_query('mod_trial_products', [
            'product_id' => $_POST['product_id'],
            'trial_days' => $_POST['trial_days'],
            'trial_price' => $_POST['trial_price'] ?: 0,
            'convert_to_product_id' => $_POST['convert_product'],
            'convert_price' => $_POST['convert_price']
        ]);
        echo '<div class="alert alert-success">Trial product configured!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Configure Trial Product</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Trial Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-4"><div class="form-group"><label>Trial Days</label>
          <input type="number" name="trial_days" class="form-control" value="14" required /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Trial Price ($0 = free)</label>
          <input type="number" step="0.01" name="trial_price" class="form-control" value="0" /></div></div>
          </div>
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Convert To (Product ID)</label>
          <input type="number" name="convert_product" class="form-control" /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Conversion Price</label>
          <input type="number" step="0.01" name="convert_price" class="form-control" /></div></div>
          </div>
          <button type="submit" name="configure_trial" class="btn btn-primary">Save</button>
          </div></form>';
    
    $trials = select_query("mod_trial_products", "*", "", "id");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>Trial Days</th><th>Trial Price</th><th>Convert To</th></tr></thead>
          <tbody>';
    while ($t = mysql_fetch_array($trials)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $t['product_id']]));
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $t['trial_days'] . '</td>
              <td>$' . $t['trial_price'] . '</td>
              <td>' . ($t['convert_to_product_id'] ?: 'N/A') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ServiceCreation', 1, function($vars) {
    $trial = mysql_fetch_array(select_query("mod_trial_products", "*", ["product_id" => $vars['pid']]));
    if ($trial) {
        $start = date('Y-m-d');
        $end = date('Y-m-d', strtotime('+' . $trial['trial_days'] . ' days'));
        insert_query("mod_trial_instances", [
            'service_id' => $vars['service_id'],
            'trial_start' => $start,
            'trial_end' => $end
        ]);
        return ['trial_active' => true, 'trial_end' => $end];
    }
});
```