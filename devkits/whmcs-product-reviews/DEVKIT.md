# WHMCS Product Reviews - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_product_reviews_config() {
    return [
        'name' => 'Product Reviews',
        'description' => 'Product review system with ratings and moderation',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'require_purchase' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Require purchase to review'],
            'moderate_reviews' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Moderate reviews before publishing']
        ]
    ];
}

function whmcs_product_reviews_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_product_reviews` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `user_id` INT(11) NOT NULL,
        `rating` TINYINT(1) NOT NULL,
        `review_title` VARCHAR(255) NOT NULL,
        `review_text` TEXT NOT NULL,
        `is_verified` TINYINT(1) DEFAULT 0,
        `helpful_yes` INT(11) DEFAULT 0,
        `helpful_no` INT(11) DEFAULT 0,
        `status` ENUM('pending','approved','rejected') DEFAULT 'pending',
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_product_reviews_deactivate() { return ['status' => 'success']; }

function whmcs_product_reviews_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Product Reviews</h2>';
    
    // Handle status changes
    if ($_POST['update_status']) {
        update_query('mod_product_reviews', ['status' => $_POST['status']], ['id' => $_POST['review_id']]);
        echo '<div class="alert alert-success">Review status updated!</div>';
    }
    
    // List pending reviews
    $pending = select_query("mod_product_reviews", "*", ["status" => "pending"], "created_at", "DESC");
    $count = mysql_num_rows($pending);
    
    echo '<div class="panel panel-default">
          <div class="panel-heading">Pending Reviews (' . $count . ')</div>
          <div class="panel-body">';
    
    if ($count > 0) {
        echo '<table class="datatable">
              <thead><tr><th>Product</th><th>User</th><th>Rating</th><th>Title</th><th>Date</th><th>Actions</th></tr></thead>
              <tbody>';
        while ($r = mysql_fetch_array($pending)) {
            $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $r['product_id']]));
            $u = mysql_fetch_array(select_query("tblclients", "firstname,lastname", ["id" => $r['user_id']]));
            echo '<tr>
                <td>' . ($p['name'] ?? 'N/A') . '</td>
                <td>' . $u['firstname'] . ' ' . substr($u['lastname'], 0, 1) . '.</td>
                <td>' . str_repeat('★', $r['rating']) . '</td>
                <td>' . $r['review_title'] . '</td>
                <td>' . $r['created_at'] . '</td>
                <td>
                <form method="post" style="display:inline;">
                <input type="hidden" name="review_id" value="' . $r['id'] . '" />
                <select name="status" style="display:inline;width:auto;" class="form-control">
                <option value="approved">Approve</option>
                <option value="rejected">Reject</option>
                </select>
                <button type="submit" name="update_status" class="btn btn-xs btn-primary">Update</button>
                </form>
                </td></tr>';
        }
        echo '</tbody></table>';
    } else {
        echo '<p>No pending reviews.</p>';
    }
    echo '</div></div>';
    
    // Stats
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, AVG(rating) as avg_rating FROM mod_product_reviews WHERE status='approved'"));
    echo '<div class="row" style="margin-top:20px;">
          <div class="col-md-6"><div class="panel panel-info">
          <div class="panel-heading">Statistics</div>
          <div class="panel-body">
          <p>Total Reviews: ' . $stats['total'] . '</p>
          <p>Average Rating: ' . round($stats['avg_rating'], 1) . '/5</p>
          </div></div></div></div>';
}

add_hook('ProductDetailsPreOutput', 1, function($vars) {
    $reviews = select_query("mod_product_reviews", "*", ["product_id" => $vars['pid'], "status" => "approved"], "created_at", "DESC");
    $items = [];
    while ($r = mysql_fetch_array($reviews)) {
        $items[] = $r;
    }
    $avg = mysql_fetch_array(full_query("SELECT AVG(rating) as avg FROM mod_product_reviews WHERE product_id=" . (int)$vars['pid'] . " AND status='approved'"));
    return ['reviews' => $items, 'average_rating' => round($avg['avg'] ?? 0, 1)];
});
```