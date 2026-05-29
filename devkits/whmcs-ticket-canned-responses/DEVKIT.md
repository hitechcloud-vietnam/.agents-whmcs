# WHMCS Ticket Canned Responses - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_canned_responses_config() { return ['name' => 'Canned Responses', 'description' => 'Pre-defined ticket responses', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_canned_responses_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_canned_responses` (`id` INT(11) NOT NULL AUTO_INCREMENT, `title` VARCHAR(255) NOT NULL, `category` VARCHAR(100) DEFAULT 'General', `content` TEXT NOT NULL, `shortcut` VARCHAR(50), `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_canned_responses_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_canned_responses_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Canned Responses</h2>';
    if ($_POST['add_response']) {
        insert_query('mod_canned_responses', ['title' => $_POST['title'], 'category' => $_POST['category'], 'content' => $_POST['content'], 'shortcut' => $_POST['shortcut']]);
        echo '<div class="alert alert-success">Response added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Canned Response</div><div class="panel-body">
          <div class="form-group"><label>Title</label><input type="text" name="title" class="form-control" required /></div>
          <div class="form-group"><label>Category</label><input type="text" name="category" class="form-control" value="General" /></div>
          <div class="form-group"><label>Shortcut</label><input type="text" name="shortcut" class="form-control" placeholder="/greeting" /></div>
          <div class="form-group"><label>Content</label><textarea name="content" class="form-control" rows="5" required></textarea></div>
          <button type="submit" name="add_response" class="btn btn-primary">Add Response</button></div></form>';
    
    $responses = select_query('mod_canned_responses', '*', '', 'category, title');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Title</th><th>Category</th><th>Shortcut</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($responses)) { echo '<tr><td>' . $r['title'] . '</td><td>' . $r['category'] . '</td><td>' . $r['shortcut'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```