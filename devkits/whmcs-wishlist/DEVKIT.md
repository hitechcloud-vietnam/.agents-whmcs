# WHMCS Wishlist - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_wishlist_config() {
    return [
        'name' => 'Wishlist',
        'description' => 'Customer wishlist with sharing and price alerts',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'enable_sharing' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable wishlist sharing'],
            'price_alerts' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable price drop alerts'],
            'stock_alerts' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable stock notifications']
        ]
    ];
}

function whmcs_wishlist_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_wishlist` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `product_id` INT(11) NOT NULL,
        `note` TEXT,
        `priority` INT(11) DEFAULT 0,
        `price_at_add` DECIMAL(10,2) DEFAULT NULL,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `user_id` (`user_id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_wishlist_alerts` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `wishlist_id` INT(11) NOT NULL,
        `alert_type` ENUM('price_drop','back_in_stock') NOT NULL,
        `sent_at` DATETIME DEFAULT NULL,
        `is_sent` TINYINT(1) DEFAULT 0,
        PRIMARY KEY (`id`),
        KEY `wishlist_id` (`wishlist_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_wishlist_shares` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `share_code` VARCHAR(50) NOT NULL UNIQUE,
        `is_public` TINYINT(1) DEFAULT 1,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `share_code` (`share_code`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_wishlist_deactivate() { return ['status' => 'success']; }

function whmcs_wishlist_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Wishlist Management</h2>';
    
    // Stats
    $total = mysql_fetch_array(full_query("SELECT COUNT(*) as count FROM mod_wishlist"));
    $shared = mysql_fetch_array(full_query("SELECT COUNT(*) as count FROM mod_wishlist_shares"));
    
    echo '<div class="row">
          <div class="col-md-6"><div class="panel panel-info">
          <div class="panel-heading">Statistics</div>
          <div class="panel-body">
          <p>Total Wishlist Items: ' . $total['count'] . '</p>
          <p>Shared Wishlists: ' . $shared['count'] . '</p>
          </div></div></div></div>';
    
    // Recent wishlists
    $recent = select_query("mod_wishlist", "*", "", "created_at", "DESC", "20");
    echo '<div class="panel panel-default">
          <div class="panel-heading">Recent Wishlist Activity</div>
          <div class="panel-body">
          <table class="datatable">
          <thead><tr><th>User</th><th>Product</th><th>Added</th><th>Price at Add</th></tr></thead>
          <tbody>';
    while ($w = mysql_fetch_array($recent)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $w['product_id']]));
        $u = mysql_fetch_array(select_query("tblclients", "email", ["id" => $w['user_id']]));
        echo '<tr><td>' . substr($u['email'], 0, strpos($u['email'], '@')) . '</td>
              <td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $w['created_at'] . '</td>
              <td>' . ($w['price_at_add'] ? '$' . $w['price_at_add'] : 'N/A') . '</td></tr>';
    }
    echo '</tbody></table></div></div>';
}

// API functions
function addToWishlist($userId, $productId, $note = '') {
    $product = mysql_fetch_array(select_query("tbld products", "pricing", ["id" => $productId]));
    $pricing = json_decode($product['pricing'], true);
    $price = $pricing['monthly'] ?? 0;
    
    insert_query('mod_wishlist', [
        'user_id' => $userId,
        'product_id' => $productId,
        'note' => $note,
        'price_at_add' => $price
    ]);
    return true;
}

function getUserWishlist($userId) {
    return select_query("mod_wishlist", "*", ["user_id" => $userId], "priority, created_at", "DESC");
}

function shareWishlist($userId) {
    $code = strtoupper(substr(md5(uniqid()), 0, 10));
    insert_query('mod_wishlist_shares', ['user_id' => $userId, 'share_code' => $code]);
    return $code;
}

function getSharedWishlist($code) {
    $share = mysql_fetch_array(select_query("mod_wishlist_shares", "*", ["share_code" => $code]));
    if ($share) {
        return getUserWishlist($share['user_id']);
    }
    return null;
}

add_hook('AddToWishlist', 1, function($vars) {
    return addToWishlist($_SESSION['uid'], $vars['pid'], $vars['note'] ?? '');
});
```