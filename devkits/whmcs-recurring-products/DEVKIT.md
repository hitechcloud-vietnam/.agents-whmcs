# WHMCS Recurring Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_recurring_products_config() {
    return [
        'name' => 'Recurring Products',
        'description' => 'Advanced recurring billing with custom cycles and proration',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'enable_proration' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable proration'],
            'grace_period' => ['Type' => 'text', 'Default' => '3', 'Description' => 'Grace period (days)']
        ]
    ];
}

function whmcs_recurring_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_recurring_custom_cycles` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `cycle_days` INT(11) NOT NULL,
        `cycle_name` VARCHAR(100) NOT NULL,
        `price` DECIMAL(10,2) NOT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_recurring_renewals` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `service_id` INT(11) NOT NULL,
        `next_date` DATE NOT NULL,
        `grace_end_date` DATE DEFAULT NULL,
        `status` ENUM('active','grace','pending','cancelled') DEFAULT 'active',
        PRIMARY KEY (`id`),
        KEY `service_id` (`service_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_recurring_products_deactivate() { return ['status' => 'success']; }

function whmcs_recurring_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Recurring Products</h2>';
    
    if ($_POST['add_cycle']) {
        insert_query('mod_recurring_custom_cycles', [
            'product_id' => $_POST['product_id'],
            'cycle_days' => $_POST['cycle_days'],
            'cycle_name' => $_POST['cycle_name'],
            'price' => $_POST['price']
        ]);
        echo '<div class="alert alert-success">Custom cycle added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Custom Billing Cycle</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>Cycle Name</label>
          <input type="text" name="cycle_name" class="form-control" placeholder="e.g., 6 months" required /></div></div>
          </div>
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Cycle Days</label>
          <input type="number" name="cycle_days" class="form-control" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Price</label>
          <input type="number" step="0.01" name="price" class="form-control" required /></div></div>
          </div>
          <button type="submit" name="add_cycle" class="btn btn-primary">Add Cycle</button>
          </div></form>';
    
    $cycles = select_query("mod_recurring_custom_cycles", "*", "", "id");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>Cycle</th><th>Days</th><th>Price</th></tr></thead>
          <tbody>';
    while ($c = mysql_fetch_array($cycles)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $c['product_id']]));
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $c['cycle_name'] . '</td>
              <td>' . $c['cycle_days'] . '</td>
              <td>$' . $c['price'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('Service Renewal', 1, function($vars) {
    $renewal = mysql_fetch_array(select_query("mod_recurring_renewals", "*", ["service_id" => $vars['service_id']]));
    if ($renewal) {
        return ['next_date' => $renewal['next_date'], 'status' => $renewal['status']];
    }
});

function calculateProration($originalPrice, $daysUsed, $totalDays) {
    $dailyRate = $originalPrice / $totalDays;
    $unusedDays = $totalDays - $daysUsed;
    return ['credit' => $dailyRate * $unusedDays, 'charge' => $dailyRate * $daysUsed];
}
```