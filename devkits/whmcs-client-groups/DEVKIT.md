# WHMCS Client Groups - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_groups_config() {
    return [
        'name' => 'Client Groups',
        'description' => 'Advanced client grouping with custom rules',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_assign' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-assign based on rules'],
            'group_pricing' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Enable group-based pricing']
        ]
    ];
}

function whmcs_client_groups_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_groups` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `group_name` VARCHAR(255) NOT NULL,
        `group_color` VARCHAR(7) DEFAULT '#007bff',
        `discount_percent` DECIMAL(5,2) DEFAULT 0.00,
        `auto_rules` TEXT,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_group_members` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `group_id` INT(11) NOT NULL,
        `assigned_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        UNIQUE KEY `user_id` (`user_id`, `group_id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_client_groups_deactivate() { return ['status' => 'success']; }

function whmcs_client_groups_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Groups</h2>';
    
    if ($_POST['create_group']) {
        insert_query('mod_client_groups', [
            'group_name' => $_POST['group_name'],
            'group_color' => $_POST['group_color'] ?: '#007bff',
            'discount_percent' => $_POST['discount'] ?: 0,
            'auto_rules' => json_encode($_POST['rules'] ?? [])
        ]);
        echo '<div class="alert alert-success">Group created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Client Group</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Group Name</label>
          <input type="text" name="group_name" class="form-control" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Color</label>
          <input type="color" name="group_color" class="form-control" value="#007bff" /></div></div>
          </div>
          <div class="form-group"><label>Discount %</label>
          <input type="number" step="0.01" name="discount" class="form-control" /></div>
          <button type="submit" name="create_group" class="btn btn-primary">Create Group</button>
          </div></form>';
    
    $groups = select_query('mod_client_groups', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Group</th><th>Color</th><th>Discount</th><th>Members</th></tr></thead>
          <tbody>';
    while ($g = mysql_fetch_array($groups)) {
        $count = mysql_num_rows(select_query('mod_client_group_members', 'id', ['group_id' => $g['id']]));
        echo '<tr><td>' . $g['group_name'] . '</td>
              <td><div style="width:20px;height:20px;background:' . $g['group_color'] . ';border-radius:3px;"></div></td>
              <td>' . $g['discount_percent'] . '%</td>
              <td>' . $count . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function assignClientToGroup($userId, $groupId) {
    insert_query('mod_client_group_members', ['user_id' => $userId, 'group_id' => $groupId]);
}

function getClientGroup($userId) {
    $result = mysql_fetch_array(select_query('mod_client_group_members m', 
        'g.*', ["m.user_id" => $userId], '', '1', '', 'm.id', 
        'INNER JOIN mod_client_groups g ON m.group_id = g.id'));
    return $result;
}

function getGroupDiscount($userId) {
    $group = getClientGroup($userId);
    return $group ? $group['discount_percent'] : 0;
}

add_hook('ClientRegistration', 1, function($vars) {
    $rules = select_query('mod_client_groups', '*', ['is_active' => 1]);
    while ($g = mysql_fetch_array($rules)) {
        $autoRules = json_decode($g['auto_rules'] ?? '[]', true);
        if (empty($autoRules)) continue;
        
        $match = true;
        foreach ($autoRules as $rule) {
            if ($rule['field'] == 'email' && strpos($vars['email'], $rule['value']) === false) {
                $match = false;
            }
        }
        if ($match) {
            assignClientToGroup($vars['user_id'], $g['id']);
        }
    }
});
```