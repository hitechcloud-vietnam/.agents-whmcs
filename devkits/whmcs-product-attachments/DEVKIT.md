# WHMCS Product Attachments - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_product_attachments_config() {
    return [
        'name' => 'Product Attachments',
        'description' => 'Manage product attachments with versioning and tracking',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_deliver' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-deliver on purchase'],
            'track_downloads' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Track download analytics']
        ]
    ];
}

function whmcs_product_attachments_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_product_attachments` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `file_name` VARCHAR(255) NOT NULL,
        `file_path` VARCHAR(500) NOT NULL,
        `file_size` INT(11) DEFAULT 0,
        `version` VARCHAR(20) DEFAULT '1.0',
        `is_active` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_attachment_downloads` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `attachment_id` INT(11) NOT NULL,
        `user_id` INT(11) DEFAULT NULL,
        `service_id` INT(11) DEFAULT NULL,
        `download_count` INT(11) DEFAULT 1,
        `last_download` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `attachment_id` (`attachment_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_product_attachments_deactivate() { return ['status' => 'success']; }

function whmcs_product_attachments_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Product Attachments</h2>';
    
    if ($_POST['upload_attachment']) {
        insert_query('mod_product_attachments', [
            'product_id' => $_POST['product_id'],
            'file_name' => $_POST['file_name'],
            'file_path' => $_POST['file_path'],
            'file_size' => $_POST['file_size'] ?: 0,
            'version' => $_POST['version'] ?: '1.0'
        ]);
        echo '<div class="alert alert-success">Attachment added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Attachment</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>File Name</label>
          <input type="text" name="file_name" class="form-control" required /></div></div>
          </div>
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>File Path</label>
          <input type="text" name="file_path" class="form-control" placeholder="/path/to/file" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Version</label>
          <input type="text" name="version" class="form-control" value="1.0" /></div></div>
          </div>
          <div class="form-group"><label>File Size (bytes)</label>
          <input type="number" name="file_size" class="form-control" /></div>
          <button type="submit" name="upload_attachment" class="btn btn-primary">Add Attachment</button>
          </div></form>';
    
    $attachments = select_query("mod_product_attachments", "*", "", "id");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>File</th><th>Version</th><th>Size</th><th>Downloads</th></tr></thead>
          <tbody>';
    while ($a = mysql_fetch_array($attachments)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $a['product_id']]));
        $dl = mysql_fetch_array(select_query("mod_attachment_downloads", "SUM(download_count) as total", ["attachment_id" => $a['id']]));
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $a['file_name'] . '</td>
              <td>' . $a['version'] . '</td>
              <td>' . number_format($a['file_size'] / 1024, 1) . ' KB</td>
              <td>' . ($dl['total'] ?: 0) . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('OrderFulfillment', 1, function($vars) {
    $attachments = select_query("mod_product_attachments", "*", ["product_id" => $vars['pid'], "is_active" => 1]);
    $files = [];
    while ($a = mysql_fetch_array($attachments)) {
        $files[] = ['name' => $a['file_name'], 'path' => $a['file_path'], 'version' => $a['version']];
    }
    if (!empty($files)) {
        return ['attachments' => $files];
    }
});
```