# WHMCS KB Rating - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_kb_rating_config() { return ['name' => 'KB Rating', 'description' => 'Article ratings', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_kb_rating_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_kb_ratings` (`id` INT(11) NOT NULL AUTO_INCREMENT, `article_id` INT(11) NOT NULL, `user_id` INT(11) DEFAULT NULL, `rating` TINYINT(1) NOT NULL, `helpful` TINYINT(1) DEFAULT 0, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_kb_rating_deactivate() { return ['status' => 'success']; }

function whmcs_kb_rating_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>KB Ratings</h2>';
    $stats = mysql_fetch_array(full_query("SELECT AVG(rating) as avg_rating, SUM(helpful) as helpful_count FROM mod_kb_ratings"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Avg Rating: ' . round($stats['avg_rating'] ?: 0, 1) . '/5</p><p>Helpful Votes: ' . $stats['helpful_count'] . '</p></div></div>';
    
    $ratings = select_query('mod_kb_ratings', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable"><thead><tr><th>Article</th><th>Rating</th><th>Helpful</th><th>Date</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($ratings)) {
        $a = mysql_fetch_array(select_query('mod_kb_articles', 'title', ['id' => $r['article_id']]));
        echo '<tr><td>' . ($a['title'] ?? 'N/A') . '</td><td>' . str_repeat('★', $r['rating']) . '</td><td>' . ($r['helpful'] ? 'Yes' : 'No') . '</td><td>' . $r['created_at'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function rateArticle($articleId, $rating, $helpful = 0) {
    insert_query('mod_kb_ratings', ['article_id' => $articleId, 'user_id' => $_SESSION['uid'], 'rating' => $rating, 'helpful' => $helpful]);
    update_query('mod_kb_articles', ['rating' => "SELECT AVG(rating) FROM mod_kb_ratings WHERE article_id=" . (int)$articleId], ['id' => $articleId]);
}
```