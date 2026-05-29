# WHMCS Client Tags - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_tags_config() {
    return [
        'name' => 'Client Tags',
        'description' => 'Tag management for clients with auto-tagging rules',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_tag' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable auto-tagging'],
            'show_in_portal' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show tags in portal']
        ]
    ];
}

function whmcs_client_tags_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_tags` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `tag_name` VARCHAR(100) NOT NULL,
        `tag_color` VARCHAR(7) DEFAULT '#6c757d',
        `tag_category` VARCHAR(100) DEFAULT NULL,
        `auto_rules` TEXT,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_tag_assignments` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `tag_id` INT(11) NOT NULL,
        `assigned_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        UNIQUE KEY `user_tag` (`user_id`, `tag_id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_client_tags_deactivate() { return ['status' => 'success']; }

function whmcs_client_tags_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Tags</h2>';
    
    if ($_POST['create_tag']) {
        insert_query('mod_client_tags', [
            'tag_name' => $_POST['tag_name'],
            'tag_color' => $_POST['tag_color'] ?: '#6c757d',
            'tag_category' => $_POST['category']
        ]);
        echo '<div class="alert alert-success">Tag created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Tag</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Tag Name</label>
          <input type="text" name="tag_name" class="form-control" required /></div></div>
          <div class="col-md-6"><div class="form-group"><label>Color</label>
          <input type="color" name="tag_color" class="form-control" value="#6c757d" /></div></div>
          </div>
          <div class="form-group"><label>Category</label>
          <input type="text" name="category" class="form-control" /></div>
          <button type="submit" name="create_tag" class="btn btn-primary">Create Tag</button>
          </div></form>';
    
    $tags = select_query('mod_client_tags', '*', '', 'tag_name');
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Tag</th><th>Color</th><th>Category</th><th>Clients</th></tr></thead>
          <tbody>';
    while ($t = mysql_fetch_array($tags)) {
        $count = mysql_num_rows(select_query('mod_client_tag_assignments', 'id', ['tag_id' => $t['id']]));
        echo '<tr><td><span style="background:' . $t['tag_color'] . ';color:white;padding:2px 8px;border-radius:10px;">' . $t['tag_name'] . '</span></td>
              <td>' . $t['tag_color'] . '</td>
              <td>' . ($t['tag_category'] ?: 'None') . '</td>
              <td>' . $count . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function assignClientTag($userId, $tagId) {
    insert_query('mod_client_tag_assignments', ['user_id' => $userId, 'tag_id' => $tagId]);
}

function getClientTags($userId) {
    return select_query('mod_client_tag_assignments ta', 't.*', ['ta.user_id' => $userId], '',
        '', '', 't.tag_name', 'INNER JOIN mod_client_tags t ON ta.tag_id = t.id');
}

add_hook('ClientCreation', 1, function($vars) {
    $rules = select_query('mod_client_tags', '*', ['is_active' => 1]);
    while ($tag = mysql_fetch_array($rules)) {
        $autoRules = json_decode($tag['auto_rules'] ?? '[]', true);
        if (empty($autoRules)) continue;
        
        $match = true;
        foreach ($autoRules as $rule) {
            $client = mysql_fetch_array(select_query('tblclients', $rule['field'], ['id' => $vars['userid']]));
            if ($client && strpos($client[$rule['field']], $rule['value']) === false) {
                $match = false;
            }
        }
        if ($match) {
            assignClientTag($vars['userid'], $tag['id']);
        }
    }
});
```