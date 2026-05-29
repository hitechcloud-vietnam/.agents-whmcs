# WHMCS Downloads - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_downloads_config() { return ['name' => 'Downloads', 'description' => 'File download management', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_downloads_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_downloads` (`id` INT(11) NOT NULL AUTO_INCREMENT, `category_id` INT(11) NOT NULL, `title` VARCHAR(255) NOT NULL, `filename` VARCHAR(255), `file_path` VARCHAR(500), `file_size` INT(11) DEFAULT 0, `download_count` INT(11) DEFAULT 0, `version` VARCHAR(20) DEFAULT '1.0', `is_active` TINYINT(1) DEFAULT 1, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_download_categories` (`id` INT(11) NOT NULL AUTO_INCREMENT, `name` VARCHAR(255) NOT NULL, `description` TEXT, `sort_order` INT(11) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_downloads_deactivate() { return ['status' => 'success']; }

function whmcs_downloads_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Downloads</h2>';
    if ($_POST['add_download']) {
        insert_query('mod_downloads', ['category_id' => $_POST['category_id'], 'title' => $_POST['title'], 'filename' => $_POST['filename'], 'file_path' => $_POST['file_path'], 'version' => $_POST['version']]);
        echo '<div class="alert alert-success">Download added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Download</div><div class="panel-body">
          <div class="form-group"><label>Category</label><select name="category_id" class="form-control">';
    $cats = select_query('mod_download_categories', '*', '', 'name');
    while ($c = mysql_fetch_array($cats)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Title</label><input type="text" name="title" class="form-control" required /></div>
          <div class="form-group"><label>File Path</label><input type="text" name="file_path" class="form-control" /></div>
          <div class="form-group"><label>Version</label><input type="text" name="version" class="form-control" value="1.0" /></div>
          <button type="submit" name="add_download" class="btn btn-primary">Add</button></div></form>';
    
    $downloads = select_query('mod_downloads', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Title</th><th>Version</th><th>Downloads</th><th>Active</th></tr></thead><tbody>';
    while ($d = mysql_fetch_array($downloads)) { echo '<tr><td>' . $d['title'] . '</td><td>' . $d['version'] . '</td><td>' . $d['download_count'] . '</td><td>' . ($d['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```