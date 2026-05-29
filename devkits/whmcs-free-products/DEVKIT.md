# WHMCS Free Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_free_products_config() {
    return [
        'name' => 'Free Products',
        'description' => 'Manage free products with optional upsells',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'show_upsells' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show upsell options'],
            'download_limits' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable download limits']
        ]
    ];
}

function whmcs_free_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_free_products` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `max_downloads` INT(11) DEFAULT NULL,
        `upsell_product_id` INT(11) DEFAULT NULL,
        `upsell_discount` DECIMAL(5,2) DEFAULT 0.00,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_free_downloads` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `service_id` INT(11) NOT NULL,
        `product_id` INT(11) NOT NULL,
        `download_count` INT(11) DEFAULT 0,
        `last_download` DATETIME DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `service_id` (`service_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_free_products_deactivate() { return ['status' => 'success']; }

function whmcs_free_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Free Products</h2>';
    
    if ($_POST['configure_free']) {
        insert_query('mod_free_products', [
            'product_id' => $_POST['product_id'],
            'max_downloads' => $_POST['max_downloads'] ?: null,
            'upsell_product_id' => $_POST['upsell_product'],
            'upsell_discount' => $_POST['upsell_discount'] ?: 0
        ]);
        echo '<div class="alert alert-success">Free product configured!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Configure Free Product</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>Max Downloads (leave empty for unlimited)</label>
          <input type="number" name="max_downloads" class="form-control" /></div></div>
          </div>
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Upsell Product ID</label>
          <input type="number" name="upsell_product" class="form-control" /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Upsell Discount %</label>
          <input type="number" step="0.01" name="upsell_discount" class="form-control" /></div></div>
          </div>
          <button type="submit" name="configure_free" class="btn btn-primary">Save</button>
          </div></form>';
}

add_hook('ProductDownload', 1, function($vars) {
    $free = mysql_fetch_array(select_query("mod_free_products", "*", ["product_id" => $vars['pid']]));
    if ($free && $free['max_downloads']) {
        $dl = mysql_fetch_array(select_query("mod_free_downloads", "*", 
            ["service_id" => $vars['service_id'], "product_id" => $vars['pid']]));
        if ($dl) {
            if ($dl['download_count'] >= $free['max_downloads']) {
                return ['allowed' => false, 'error' => 'Download limit reached'];
            }
            update_query("mod_free_downloads", ["download_count" => "+=1", "last_download" => date('Y-m-d H:i:s')], 
                ["id" => $dl['id']]);
        } else {
            insert_query("mod_free_downloads", [
                "service_id" => $vars['service_id'],
                "product_id" => $vars['pid'],
                "download_count" => 1,
                "last_download" => date('Y-m-d H:i:s')
            ]);
        }
    }
    return ['allowed' => true];
});
```