# WHMCS One-Time Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_one_time_products_config() {
    return [
        'name' => 'One-Time Products',
        'description' => 'Manage one-time purchase products with instant delivery',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_deliver' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-deliver after payment'],
            'generate_license' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Generate license keys']
        ]
    ];
}

function whmcs_one_time_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_onetime_products` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `setup_fee` DECIMAL(10,2) DEFAULT 0.00,
        `license_prefix` VARCHAR(50) DEFAULT NULL,
        `delivery_file` VARCHAR(255) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_license_keys` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `order_id` INT(11) NOT NULL,
        `license_key` VARCHAR(255) NOT NULL,
        `issued_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `order_id` (`order_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_one_time_products_deactivate() { return ['status' => 'success']; }

function whmcs_one_time_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>One-Time Products</h2>';
    
    if ($_POST['configure']) {
        insert_query('mod_onetime_products', [
            'product_id' => $_POST['product_id'],
            'setup_fee' => $_POST['setup_fee'] ?: 0,
            'license_prefix' => $_POST['license_prefix'],
            'delivery_file' => $_POST['delivery_file']
        ]);
        echo '<div class="alert alert-success">Product configured!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Configure One-Time Product</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>Setup Fee</label>
          <input type="number" step="0.01" name="setup_fee" class="form-control" /></div></div>
          </div>
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>License Prefix</label>
          <input type="text" name="license_prefix" class="form-control" placeholder="e.g., PRO-" /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Delivery File</label>
          <input type="text" name="delivery_file" class="form-control" placeholder="path/to/file" /></div></div>
          </div>
          <button type="submit" name="configure" class="btn btn-primary">Save Configuration</button>
          </div></form>';
    
    $configs = select_query("mod_onetime_products", "*", "", "id");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>Setup Fee</th><th>License Prefix</th><th>File</th></tr></thead>
          <tbody>';
    while ($c = mysql_fetch_array($configs)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $c['product_id']]));
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>$' . $c['setup_fee'] . '</td>
              <td>' . $c['license_prefix'] . '</td>
              <td>' . $c['delivery_file'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function generateLicenseKey($prefix = '') {
    $key = strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 4)) . '-' .
           strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 4)) . '-' .
           strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 4)) . '-' .
           strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 4));
    return $prefix . $key;
}

add_hook('OrderFulfillment', 1, function($vars) {
    $config = mysql_fetch_array(select_query("mod_onetime_products", "*", ["product_id" => $vars['pid']]));
    if ($config) {
        if ($config['license_prefix']) {
            $key = generateLicenseKey($config['license_prefix']);
            insert_query("mod_license_keys", [
                "product_id" => $vars['pid'],
                "order_id" => $vars['order_id'],
                "license_key" => $key
            ]);
            return ['license_key' => $key];
        }
    }
});
```