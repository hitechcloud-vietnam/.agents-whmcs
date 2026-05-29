# WHMCS Addon Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_addon_products_config() {
    return [
        'name' => 'Addon Products',
        'description' => 'Manage product addons with recommendations and dependencies',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_recommend' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-recommend addons'],
            'allow_multiple' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow multiple addons']
        ]
    ];
}

function whmcs_addon_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_product_addons` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `addon_name` VARCHAR(255) NOT NULL,
        `addon_description` TEXT,
        `price` DECIMAL(10,2) NOT NULL,
        `billing_cycle` ENUM('monthly','quarterly','annually','biennially') DEFAULT 'monthly',
        `product_id` INT(11) DEFAULT NULL,
        `category` VARCHAR(100) DEFAULT NULL,
        `is_required` TINYINT(1) DEFAULT 0,
        `is_active` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_addon_dependencies` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `addon_id` INT(11) NOT NULL,
        `requires_addon_id` INT(11) NOT NULL,
        PRIMARY KEY (`id`),
        KEY `addon_id` (`addon_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_addon_products_deactivate() { return ['status' => 'success']; }

function whmcs_addon_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Addon Products</h2>';
    
    if ($_POST['create_addon']) {
        insert_query('mod_product_addons', [
            'addon_name' => $_POST['addon_name'],
            'addon_description' => $_POST['addon_description'],
            'price' => $_POST['price'],
            'billing_cycle' => $_POST['billing_cycle'],
            'product_id' => $_POST['product_id'] ?: null,
            'category' => $_POST['category']
        ]);
        echo '<div class="alert alert-success">Addon created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Addon</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Addon Name</label>
          <input type="text" name="addon_name" class="form-control" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Category</label>
          <input type="text" name="category" class="form-control" /></div></div>
          </div>
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Price</label>
          <input type="number" step="0.01" name="price" class="form-control" required /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Billing Cycle</label>
          <select name="billing_cycle" class="form-control">
          <option value="monthly">Monthly</option>
          <option value="quarterly">Quarterly</option>
          <option value="annually">Annually</option>
          </select></div></div>
          <div class="col-md-4"><div class="form-group"><label>Required</label>
          <select name="is_required" class="form-control">
          <option value="0">Optional</option>
          <option value="1">Required</option>
          </select></div></div>
          </div>
          <div class="form-group"><label>Description</label>
          <textarea name="addon_description" class="form-control"></textarea></div>
          <button type="submit" name="create_addon" class="btn btn-primary">Create Addon</button>
          </div></form>';
    
    $addons = select_query("mod_product_addons", "*", "", "id", "DESC");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Name</th><th>Price</th><th>Billing</th><th>Required</th></tr></thead>
          <tbody>';
    while ($a = mysql_fetch_array($addons)) {
        echo '<tr><td>' . $a['addon_name'] . '</td>
              <td>$' . $a['price'] . '</td>
              <td>' . ucfirst($a['billing_cycle']) . '</td>
              <td>' . ($a['is_required'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ProductAddonDisplay', 1, function($vars) {
    $addons = select_query("mod_product_addons", "*", ["product_id" => $vars['pid'], "is_active" => 1]);
    $items = [];
    while ($a = mysql_fetch_array($addons)) {
        $items[] = $a;
    }
    return ['addons' => $items, 'auto_recommend' => true];
});
```