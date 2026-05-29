# WHMCS Volume Pricing - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_volume_pricing_config() {
    return [
        'name' => 'Volume Pricing',
        'description' => 'Quantity-based tiered pricing with volume discounts',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'show_table' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show pricing table'],
            'auto_apply' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-apply tier pricing']
        ]
    ];
}

function whmcs_volume_pricing_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_volume_pricing` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `min_qty` INT(11) NOT NULL,
        `max_qty` INT(11) DEFAULT NULL,
        `price_per_unit` DECIMAL(10,2) NOT NULL,
        `discount_percent` DECIMAL(5,2) DEFAULT 0.00,
        `tier_label` VARCHAR(100) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_volume_pricing_deactivate() { return ['status' => 'success']; }

function whmcs_volume_pricing_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Volume Pricing</h2>';
    
    if ($_POST['save_tier']) {
        insert_query('mod_volume_pricing', [
            'product_id' => $_POST['product_id'],
            'min_qty' => $_POST['min_qty'],
            'max_qty' => $_POST['max_qty'] ?: null,
            'price_per_unit' => $_POST['price'],
            'discount_percent' => $_POST['discount'] ?: 0,
            'tier_label' => $_POST['tier_label']
        ]);
        echo '<div class="alert alert-success">Tier saved!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Pricing Tier</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-2"><div class="form-group"><label>Min Qty</label>
          <input type="number" name="min_qty" class="form-control" required /></div></div>
          <div class="col-md-2"><div class="form-group"><label>Max Qty</label>
          <input type="number" name="max_qty" class="form-control" /></div></div>
          <div class="col-md-2"><div class="form-group"><label>Unit Price</label>
          <input type="number" step="0.01" name="price" class="form-control" required /></div></div>
          <div class="col-md-2"><div class="form-group"><label>Discount %</label>
          <input type="number" step="0.01" name="discount" class="form-control" /></div></div>
          </div>
          <div class="form-group"><label>Tier Label</label>
          <input type="text" name="tier_label" class="form-control" placeholder="e.g., Best Value" /></div>
          <button type="submit" name="save_tier" class="btn btn-primary">Save Tier</button>
          </div></form>';
    
    $tiers = select_query("mod_volume_pricing", "*", "", "product_id, min_qty");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>Qty Range</th><th>Unit Price</th><th>Discount</th><th>Label</th></tr></thead>
          <tbody>';
    while ($t = mysql_fetch_array($tiers)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $t['product_id']]));
        $qtyRange = $t['min_qty'] . ($t['max_qty'] ? '-' . $t['max_qty'] : '+');
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $qtyRange . '</td>
              <td>$' . $t['price_per_unit'] . '</td>
              <td>' . $t['discount_percent'] . '%</td>
              <td>' . $t['tier_label'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('GetProductPrice', 1, function($vars) {
    $qty = $_POST['qty'] ?? 1;
    $tier = select_query("mod_volume_pricing", "*", 
        ["product_id" => $vars['pid'], "min_qty<=" => $qty, "is_active" => 1], "min_qty", "DESC", "1");
    if ($t = mysql_fetch_array($tier)) {
        if (!$t['max_qty'] || $qty <= $t['max_qty']) {
            return ['unit_price' => $t['price_per_unit'], 'discount' => $t['discount_percent']];
        }
    }
});
```