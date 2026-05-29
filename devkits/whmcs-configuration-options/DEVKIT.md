# WHMCS Configuration Options - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_configuration_options_config() {
    return [
        'name' => 'Configuration Options',
        'description' => 'Advanced product configuration with conditional logic',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'show_prices' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show price modifiers'],
            'enable_conditionals' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable conditional logic']
        ]
    ];
}

function whmcs_configuration_options_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_config_options` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `option_name` VARCHAR(255) NOT NULL,
        `option_type` ENUM('dropdown','checkbox','radio','text','number') DEFAULT 'dropdown',
        `option_values` TEXT,
        `price_modifier` DECIMAL(10,2) DEFAULT 0.00,
        `sort_order` INT(11) DEFAULT 0,
        `is_required` TINYINT(1) DEFAULT 0,
        `depends_on` VARCHAR(255) DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_configuration_options_deactivate() { return ['status' => 'success']; }

function whmcs_configuration_options_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Configuration Options</h2>';
    
    if ($_POST['save_config']) {
        insert_query('mod_config_options', [
            'product_id' => $_POST['product_id'],
            'option_name' => $_POST['option_name'],
            'option_type' => $_POST['option_type'],
            'option_values' => json_encode($_POST['option_values']),
            'price_modifier' => $_POST['price_modifier'],
            'is_required' => $_POST['is_required'] ?? 0,
            'sort_order' => $_POST['sort_order'] ?? 0
        ]);
        echo '<div class="alert alert-success">Configuration option saved!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Configuration Option</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>Option Name</label>
          <input type="text" name="option_name" class="form-control" required /></div></div>
          </div>
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Type</label>
          <select name="option_type" class="form-control">
          <option value="dropdown">Dropdown</option>
          <option value="checkbox">Checkbox</option>
          <option value="radio">Radio Buttons</option>
          <option value="text">Text Input</option>
          <option value="number">Number Input</option>
          </select></div></div>
          <div class="col-md-4"><div class="form-group"><label>Price Modifier</label>
          <input type="number" step="0.01" name="price_modifier" class="form-control" /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Sort Order</label>
          <input type="number" name="sort_order" class="form-control" value="0" /></div></div>
          </div>
          <div class="form-group"><label>Required</label>
          <input type="checkbox" name="is_required" value="1" /></div>
          <button type="submit" name="save_config" class="btn btn-primary">Save Option</button>
          </div></form>';
    
    $options = select_query("mod_config_options", "*", "", "sort_order");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>Option</th><th>Type</th><th>Modifier</th><th>Required</th></tr></thead>
          <tbody>';
    while ($o = mysql_fetch_array($options)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $o['product_id']]));
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $o['option_name'] . '</td>
              <td>' . ucfirst($o['option_type']) . '</td>
              <td>' . ($o['price_modifier'] > 0 ? '+$' . $o['price_modifier'] : '$' . $o['price_modifier']) . '</td>
              <td>' . ($o['is_required'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ProductConfigOptionsOutput', 1, function($vars) {
    $options = select_query("mod_config_options", "*", ["product_id" => $vars['pid']], "sort_order");
    $items = [];
    while ($o = mysql_fetch_array($options)) {
        $items[] = $o;
    }
    return ['config_options' => $items, 'show_prices' => true];
});
```