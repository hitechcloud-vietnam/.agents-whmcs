# WHMCS KB Articles - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_kb_articles_config() { return ['name' => 'KB Articles', 'description' => 'Article management', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_kb_articles_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_kb_article_versions` (`id` INT(11) NOT NULL AUTO_INCREMENT, `article_id` INT(11) NOT NULL, `version` INT(11) DEFAULT 1, `title` VARCHAR(255), `content` TEXT, `created_by` INT(11), `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_kb_articles_deactivate() { return ['status' => 'success']; }

function whmcs_kb_articles_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>KB Articles</h2>';
    if ($_POST['create_article']) {
        insert_query('mod_kb_articles', ['category_id' => $_POST['category_id'], 'title' => $_POST['title'], 'content' => $_POST['content']]);
        $id = mysql_insert_id();
        insert_query('mod_kb_article_versions', ['article_id' => $id, 'title' => $_POST['title'], 'content' => $_POST['content'], 'created_by' => $_SESSION['adminid']]);
        echo '<div class="alert alert-success">Article created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Article</div><div class="panel-body">
          <div class="form-group"><label>Category</label><select name="category_id" class="form-control">';
    $cats = select_query('mod_kb_cat_hierarchy', '*', '', 'name');
    while ($c = mysql_fetch_array($cats)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Title</label><input type="text" name="title" class="form-control" required /></div>
          <div class="form-group"><label>Content (Markdown)</label><textarea name="content" class="form-control" rows="10"></textarea></div>
          <button type="submit" name="create_article" class="btn btn-primary">Create</button></div></form>';
    
    $articles = select_query('mod_kb_articles', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Title</th><th>Views</th><th>Rating</th><th>Published</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($articles)) { echo '<tr><td>' . $a['title'] . '</td><td>' . $a['views'] . '</td><td>' . $a['rating'] . '</td><td>' . ($a['is_published'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```