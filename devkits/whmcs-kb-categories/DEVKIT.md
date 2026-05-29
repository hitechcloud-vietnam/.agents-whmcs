# WHMCS KB Categories - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_kb_categories_config() { return ['name' => 'KB Categories', 'description' => 'Knowledge base categories', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_kb_categories_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_kb_cat_hierarchy` (`id` INT(11) NOT NULL AUTO_INCREMENT, `parent_id` INT(11) DEFAULT NULL, `name` VARCHAR(255) NOT NULL, `slug` VARCHAR(255), `description` TEXT, `icon` VARCHAR(50), `sort_order` INT(11) DEFAULT 0, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_kb_categories_deactivate() { return ['status' => 'success']; }

function whmcs_kb_categories_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>KB Categories</h2>';
    if ($_POST['add_category']) {
        insert_query('mod_kb_cat_hierarchy', ['parent_id' => $_POST['parent_id'] ?: null, 'name' => $_POST['name'], 'slug' => strtolower(str_replace(' ', '-', $_POST['name'])), 'sort_order' => $_POST['sort_order']]);
        echo '<div class="alert alert-success">Category created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Category</div><div class="panel-body">
          <div class="form-group"><label>Parent Category</label><select name="parent_id" class="form-control"><option value="">None (Root)</option>';
    $cats = select_query('mod_kb_cat_hierarchy', '*', '', 'name');
    while ($c = mysql_fetch_array($cats)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Name</label><input type="text" name="name" class="form-control" required /></div>
          <div class="form-group"><label>Sort Order</label><input type="number" name="sort_order" class="form-control" value="0" /></div>
          <button type="submit" name="add_category" class="btn btn-primary">Create</button></div></form>';
    
    $categories = select_query('mod_kb_cat_hierarchy', '*', '', 'sort_order');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Parent</th><th>Articles</th><th>Sort</th></tr></thead><tbody>';
    while ($c = mysql_fetch_array($categories)) {
        $parent = $c['parent_id'] ? mysql_fetch_array(select_query('mod_kb_cat_hierarchy', 'name', ['id' => $c['parent_id']])) : null;
        $count = mysql_num_rows(select_query('mod_kb_articles', 'id', ['category_id' => $c['id']]));
        echo '<tr><td>' . $c['name'] . '</td><td>' . ($parent['name'] ?? '-') . '</td><td>' . $count . '</td><td>' . $c['sort_order'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}
```