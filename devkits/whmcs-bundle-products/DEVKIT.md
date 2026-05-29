# WHMCS Bundle Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_bundle_products_config() {
    return [
        'name' => 'Bundle Products',
        'description' => 'Create and manage product bundles with combined pricing',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'show_savings' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show savings amount'],
            'allow_customize' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow bundle customization']
        ]
    ];
}

function whmcs_bundle_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_product_bundles` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `bundle_name` VARCHAR(255) NOT NULL,
        `bundle_description` TEXT,
        `bundle_price` DECIMAL(10,2) NOT NULL,
        `individual_total` DECIMAL(10,2) DEFAULT 0,
        `products` TEXT,
        `image` VARCHAR(255) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_bundle_products` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `bundle_id` INT(11) NOT NULL,
        `product_id` INT(11) NOT NULL,
        `quantity` INT(11) DEFAULT 1,
        `is_required` TINYINT(1) DEFAULT 1,
        `sort_order` INT(11) DEFAULT 0,
        PRIMARY KEY (`id`),
        KEY `bundle_id` (`bundle_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_bundle_products_deactivate() { return ['status' => 'success']; }

function whmcs_bundle_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Bundle Products</h2>';
    
    if ($_POST['create_bundle']) {
        $products = json_encode($_POST['product_ids']);
        insert_query('mod_product_bundles', [
            'bundle_name' => $_POST['bundle_name'],
            'bundle_description' => $_POST['bundle_description'],
            'bundle_price' => $_POST['bundle_price'],
            'products' => $products
        ]);
        echo '<div class="alert alert-success">Bundle created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Bundle</div>
          <div class="panel-body">
          <div class="form-group"><label>Bundle Name</label>
          <input type="text" name="bundle_name" class="form-control" required /></div>
          <div class="form-group"><label>Description</label>
          <textarea name="bundle_description" class="form-control"></textarea></div>
          <div class="form-group"><label>Bundle Price</label>
          <input type="number" step="0.01" name="bundle_price" class="form-control" required /></div>
          <div class="form-group"><label>Select Products</label>
          <select name="product_ids[]" multiple class="form-control" size="10">';
    
    $products = select_query("tbld products", "id, name", ["retired" => 0], "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    
    echo '</select></div>
          <button type="submit" name="create_bundle" class="btn btn-primary">Create Bundle</button>
          </div></form>';
    
    $bundles = select_query("mod_product_bundles", "*", "", "id", "DESC");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Name</th><th>Price</th><th>Products</th><th>Status</th></tr></thead>
          <tbody>';
    while ($b = mysql_fetch_array($bundles)) {
        $prods = json_decode($b['products'], true);
        $count = is_array($prods) ? count($prods) : 0;
        echo '<tr><td>' . $b['bundle_name'] . '</td>
              <td>$' . $b['bundle_price'] . '</td>
              <td>' . $count . '</td>
              <td>' . ($b['is_active'] ? 'Active' : 'Inactive') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('OrderFormBundleDisplay', 1, function($vars) {
    $bundles = select_query("mod_product_bundles", "*", ["is_active" => 1]);
    $data = [];
    while ($b = mysql_fetch_array($bundles)) {
        $data[] = $b;
    }
    return ['bundles' => $data, 'show_savings' => true];
});
```