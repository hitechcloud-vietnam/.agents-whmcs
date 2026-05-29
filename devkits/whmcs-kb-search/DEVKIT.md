# WHMCS KB Search - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_kb_search_config() { return ['name' => 'KB Search', 'description' => 'Advanced KB search', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'fuzzy_search' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable fuzzy search']
]];}

function whmcs_kb_search_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_kb_search_log` (`id` INT(11) NOT NULL AUTO_INCREMENT, `query` VARCHAR(255) NOT NULL, `results` INT(11) DEFAULT 0, `user_id` INT(11) DEFAULT NULL, `searched_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_kb_search_deactivate() { return ['status' => 'success']; }

function whmcs_kb_search_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>KB Search Analytics</h2>';
    
    $popular = full_query("SELECT query, COUNT(*) as searches, AVG(results) as avg_results FROM mod_kb_search_log GROUP BY query ORDER BY searches DESC LIMIT 20");
    echo '<table class="datatable"><thead><tr><th>Query</th><th>Searches</th><th>Avg Results</th></tr></thead><tbody>';
    while ($p = mysql_fetch_array($popular)) { echo '<tr><td>' . $p['query'] . '</td><td>' . $p['searches'] . '</td><td>' . round($p['avg_results']) . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function searchKB($query, $filters = []) {
    insert_query('mod_kb_search_log', ['query' => $query, 'user_id' => $_SESSION['uid'] ?? null]);
    
    $where = "title LIKE '%" . db_escape($query) . "%' OR content LIKE '%" . db_escape($query) . "%'";
    if (!empty($filters['category'])) $where .= " AND category_id = " . (int)$filters['category'];
    
    return select_query('mod_kb_articles', '*', $where . " AND is_published = 1");
}
```