# WHMCS Client Profile Manager - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_profile_manager_config() {
    return [
        'name' => 'Client Profile Manager',
        'description' => 'Enhanced profile management with custom fields and verification',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'require_verification' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Require email verification'],
            'show_completeness' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show profile completeness']
        ]
    ];
}

function whmcs_client_profile_manager_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_profile_fields` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `field_key` VARCHAR(50) NOT NULL,
        `field_label` VARCHAR(255) NOT NULL,
        `field_type` ENUM('text','select','checkbox','date','file') DEFAULT 'text',
        `options` TEXT,
        `is_required` TINYINT(1) DEFAULT 0,
        `show_in_portal` TINYINT(1) DEFAULT 1,
        `sort_order` INT(11) DEFAULT 0,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_profile_values` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `field_id` INT(11) NOT NULL,
        `field_value` TEXT,
        `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `user_id` (`user_id`),
        KEY `field_id` (`field_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_verifications` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `verification_type` VARCHAR(50) NOT NULL,
        `status` ENUM('pending','verified','rejected') DEFAULT 'pending',
        `verified_at` DATETIME DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_client_profile_manager_deactivate() { return ['status' => 'success']; }

function whmcs_client_profile_manager_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Profile Manager</h2>';
    
    if ($_POST['add_field']) {
        insert_query('mod_client_profile_fields', [
            'field_key' => strtolower(str_replace(' ', '_', $_POST['field_label'])),
            'field_label' => $_POST['field_label'],
            'field_type' => $_POST['field_type'],
            'is_required' => $_POST['is_required'] ?? 0,
            'sort_order' => $_POST['sort_order'] ?? 0
        ]);
        echo '<div class="alert alert-success">Profile field added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Custom Profile Field</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Field Label</label>
          <input type="text" name="field_label" class="form-control" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Field Type</label>
          <select name="field_type" class="form-control">
          <option value="text">Text</option>
          <option value="select">Dropdown</option>
          <option value="checkbox">Checkbox</option>
          <option value="date">Date</option>
          <option value="file">File Upload</option>
          </select></div></div>
          </div>
          <div class="form-group"><label><input type="checkbox" name="is_required" value="1" /> Required</label></div>
          <button type="submit" name="add_field" class="btn btn-primary">Add Field</button>
          </div></form>';
    
    $fields = select_query('mod_client_profile_fields', '*', '', 'sort_order');
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Label</th><th>Type</th><th>Required</th><th>Order</th></tr></thead>
          <tbody>';
    while ($f = mysql_fetch_array($fields)) {
        echo '<tr><td>' . $f['field_label'] . '</td>
              <td>' . $f['field_type'] . '</td>
              <td>' . ($f['is_required'] ? 'Yes' : 'No') . '</td>
              <td>' . $f['sort_order'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function getProfileCompleteness($userId) {
    $totalFields = mysql_num_rows(select_query('mod_client_profile_fields', 'id', ['is_required' => 1]));
    $filledFields = mysql_num_rows(select_query('mod_client_profile_values pv', 
        'pv.id', ["pv.user_id" => $userId, "pf.is_required" => 1], '',
        '', '', 'pv.id', "INNER JOIN mod_client_profile_fields pf ON pv.field_id = pf.id"));
    return $totalFields > 0 ? round(($filledFields / $totalFields) * 100) : 100;
}

add_hook('ClientAreaPageAccount', 1, function($vars) {
    $uid = $_SESSION['uid'] ?? 0;
    if (!$uid) return [];
    return ['profile_completeness' => getProfileCompleteness($uid)];
});
```