# WHMCS Client Notes - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_notes_config() {
    return [
        'name' => 'Client Notes',
        'description' => 'Enhanced client notes with categories and visibility',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'allow_templates' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow note templates'],
            'staff_only' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Notes visible to staff only']
        ]
    ];
}

function whmcs_client_notes_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_notes` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `staff_id` INT(11) DEFAULT NULL,
        `category` VARCHAR(100) DEFAULT 'general',
        `note_text` TEXT NOT NULL,
        `is_private` TINYINT(1) DEFAULT 0,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_note_templates` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `template_name` VARCHAR(255) NOT NULL,
        `template_text` TEXT NOT NULL,
        `category` VARCHAR(100) DEFAULT 'general',
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_client_notes_deactivate() { return ['status' => 'success']; }

function whmcs_client_notes_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Notes</h2>';
    
    if ($_POST['add_note']) {
        insert_query('mod_client_notes', [
            'user_id' => $_POST['user_id'],
            'staff_id' => $_SESSION['adminid'],
            'category' => $_POST['category'],
            'note_text' => $_POST['note_text'],
            'is_private' => $_POST['is_private'] ?? 0
        ]);
        echo '<div class="alert alert-success">Note added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Note</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Client</label>
          <select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname) as name', '', 'firstname');
    while ($c = mysql_fetch_array($clients)) {
        echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-4"><div class="form-group"><label>Category</label>
          <input type="text" name="category" class="form-control" value="general" /></div></div>
          <div class="col-md-4"><div class="form-group"><label><input type="checkbox" name="is_private" value="1" /> Private</label></div></div>
          </div>
          <div class="form-group"><label>Note</label>
          <textarea name="note_text" class="form-control" rows="4" required></textarea></div>
          <button type="submit" name="add_note" class="btn btn-primary">Add Note</button>
          </div></form>';
    
    $notes = select_query('mod_client_notes', '*', '', 'created_at', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Client</th><th>Category</th><th>Note</th><th>Date</th><th>Private</th></tr></thead>
          <tbody>';
    while ($n = mysql_fetch_array($notes)) {
        $c = mysql_fetch_array(select_query('tblclients', 'firstname, lastname', ['id' => $n['user_id']]));
        echo '<tr><td>' . $c['firstname'] . ' ' . $c['lastname'] . '</td>
              <td>' . $n['category'] . '</td>
              <td>' . substr($n['note_text'], 0, 50) . '...</td>
              <td>' . $n['created_at'] . '</td>
              <td>' . ($n['is_private'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ClientAreaPageAccountDetails', 1, function($vars) {
    if ($_SESSION['adminid']) {
        $notes = select_query('mod_client_notes', '*', ['user_id' => $vars['userid'], 'is_private' => 0], 'created_at', 'DESC', '10');
        $items = [];
        while ($n = mysql_fetch_array($notes)) { $items[] = $n; }
        return ['client_notes' => $items];
    }
});
```