# WHMCS Custom Products - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_custom_products_config() {
    return [
        'name' => 'Custom Products',
        'description' => 'Create custom product types with dynamic fields',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'enable_custom_types' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable custom types'],
            'custom_workflows' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable custom workflows']
        ]
    ];
}

function whmcs_custom_products_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_custom_product_types` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `type_name` VARCHAR(255) NOT NULL,
        `type_key` VARCHAR(50) NOT NULL UNIQUE,
        `fields` TEXT,
        `workflow_id` INT(11) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_custom_product_values` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `type_id` INT(11) NOT NULL,
        `field_values` TEXT,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_custom_workflows` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `workflow_name` VARCHAR(255) NOT NULL,
        `steps` TEXT,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_custom_products_deactivate() { return ['status' => 'success']; }

function whmcs_custom_products_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Custom Products</h2>';
    
    if ($_POST['create_type']) {
        insert_query('mod_custom_product_types', [
            'type_name' => $_POST['type_name'],
            'type_key' => strtolower(str_replace(' ', '_', $_POST['type_name'])),
            'fields' => json_encode($_POST['fields'])
        ]);
        echo '<div class="alert alert-success">Custom product type created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Custom Product Type</div>
          <div class="panel-body">
          <div class="form-group"><label>Type Name</label>
          <input type="text" name="type_name" class="form-control" required /></div>
          <div class="form-group"><label>Fields (JSON)</label>
          <textarea name="fields" class="form-control" rows="5" placeholder=\'[{"name":"color","type":"select","options":["red","blue"]}]\'></textarea></div>
          <button type="submit" name="create_type" class="btn btn-primary">Create Type</button>
          </div></form>';
    
    $types = select_query("mod_custom_product_types", "*", "", "id");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Type Name</th><th>Type Key</th><th>Active</th></tr></thead>
          <tbody>';
    while ($t = mysql_fetch_array($types)) {
        echo '<tr><td>' . $t['type_name'] . '</td>
              <td>' . $t['type_key'] . '</td>
              <td>' . ($t['is_active'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ProductDetailsPreOutput', 1, function($vars) {
    $custom = mysql_fetch_array(select_query("mod_custom_product_values", "*", ["product_id" => $vars['pid']]));
    if ($custom) {
        return ['custom_type' => json_decode($custom['field_values'], true)];
    }
});
```