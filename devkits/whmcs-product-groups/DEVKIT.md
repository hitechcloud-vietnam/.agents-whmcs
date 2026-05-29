# WHMCS Product Groups - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_product_groups_config() {
    return [
        'name' => 'Product Groups',
        'description' => 'Organize products into custom groups with enhanced display',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'show_group_nav' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show group navigation'],
            'featured_products' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable featured products']
        ]
    ];
}

function whmcs_product_groups_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_product_groups` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `group_name` VARCHAR(255) NOT NULL,
        `group_description` TEXT,
        `group_icon` VARCHAR(255) DEFAULT NULL,
        `group_color` VARCHAR(7) DEFAULT '#007bff',
        `sort_order` INT(11) DEFAULT 0,
        `is_featured` TINYINT(1) DEFAULT 0,
        `is_active` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_group_products` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `group_id` INT(11) NOT NULL,
        `product_id` INT(11) NOT NULL,
        `sort_order` INT(11) DEFAULT 0,
        `is_featured` TINYINT(1) DEFAULT 0,
        PRIMARY KEY (`id`),
        KEY `group_id` (`group_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_product_groups_deactivate() { return ['status' => 'success']; }

function whmcs_product_groups_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Product Groups</h2>';
    
    if ($_POST['create_group']) {
        insert_query('mod_product_groups', [
            'group_name' => $_POST['group_name'],
            'group_description' => $_POST['group_description'],
            'group_color' => $_POST['group_color'] ?: '#007bff',
            'sort_order' => $_POST['sort_order'] ?: 0,
            'is_featured' => $_POST['is_featured'] ?? 0
        ]);
        echo '<div class="alert alert-success">Group created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Product Group</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Group Name</label>
          <input type="text" name="group_name" class="form-control" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Group Color</label>
          <input type="color" name="group_color" class="form-control" value="#007bff" /></div></div>
          </div>
          <div class="form-group"><label>Description</label>
          <textarea name="group_description" class="form-control"></textarea></div>
          <div class="form-group">
          <label><input type="checkbox" name="is_featured" value="1" /> Featured Group</label>
          </div>
          <button type="submit" name="create_group" class="btn btn-primary">Create Group</button>
          </div></form>';
    
    $groups = select_query("mod_product_groups", "*", "", "sort_order");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Name</th><th>Color</th><th>Products</th><th>Featured</th></tr></thead>
          <tbody>';
    while ($g = mysql_fetch_array($groups)) {
        $count = mysql_num_rows(select_query("mod_group_products", "id", ["group_id" => $g['id']]));
        echo '<tr><td><span style="color:' . $g['group_color'] . ';">' . $g['group_name'] . '</span></td>
              <td><div style="width:20px;height:20px;background:' . $g['group_color'] . ';border-radius:3px;"></div></td>
              <td>' . $count . '</td>
              <td>' . ($g['is_featured'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ProductGroupNavigation', 1, function($vars) {
    $groups = select_query("mod_product_groups", "*", ["is_active" => 1], "sort_order");
    $items = [];
    while ($g = mysql_fetch_array($groups)) {
        $items[] = $g;
    }
    return ['groups' => $items, 'show_navigation' => true];
});
```