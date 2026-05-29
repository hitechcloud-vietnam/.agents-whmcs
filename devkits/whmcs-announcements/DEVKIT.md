# WHMCS Announcements - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_announcements_config() { return ['name' => 'Announcements', 'description' => 'Announcement management', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_announcements_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_announcements` (`id` INT(11) NOT NULL AUTO_INCREMENT, `title` VARCHAR(255) NOT NULL, `content` TEXT, `announcement_type` ENUM('info','warning','alert') DEFAULT 'info', `publish_at` DATETIME DEFAULT NULL, `end_at` DATETIME DEFAULT NULL, `target_groups` TEXT, `is_active` TINYINT(1) DEFAULT 1, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_announcements_deactivate() { return ['status' => 'success']; }

function whmcs_announcements_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Announcements</h2>';
    if ($_POST['create_announcement']) {
        insert_query('mod_announcements', ['title' => $_POST['title'], 'content' => $_POST['content'], 'announcement_type' => $_POST['type'], 'publish_at' => $_POST['publish_at'] ?: null, 'is_active' => 1]);
        echo '<div class="alert alert-success">Announcement created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Announcement</div><div class="panel-body">
          <div class="form-group"><label>Title</label><input type="text" name="title" class="form-control" required /></div>
          <div class="form-group"><label>Type</label><select name="type" class="form-control"><option value="info">Info</option><option value="warning">Warning</option><option value="alert">Alert</option></select></div>
          <div class="form-group"><label>Publish At</label><input type="datetime-local" name="publish_at" class="form-control" /></div>
          <div class="form-group"><label>Content</label><textarea name="content" class="form-control" rows="5"></textarea></div>
          <button type="submit" name="create_announcement" class="btn btn-primary">Create</button></div></form>';
    
    $anns = select_query('mod_announcements', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Title</th><th>Type</th><th>Publish</th><th>Active</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($anns)) { echo '<tr><td>' . $a['title'] . '</td><td>' . ucfirst($a['announcement_type']) . '</td><td>' . ($a['publish_at'] ?: 'Now') . '</td><td>' . ($a['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $anns = select_query('mod_announcements', '*', "is_active=1 AND (publish_at IS NULL OR publish_at <= NOW())", 'id', 'DESC', '3');
    $items = [];
    while ($a = mysql_fetch_array($anns)) { $items[] = $a; }
    return ['announcements' => $items];
});
```