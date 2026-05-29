# WHMCS Domain Marketplace - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_marketplace_config() { return ['name' => 'Domain Marketplace', 'description' => 'Domain marketplace listings', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'listing_fee' => ['Type' => 'text', 'Default' => '0', 'Description' => 'Listing fee'],
    'commission' => ['Type' => 'text', 'Default' => '10', 'Description' => 'Commission %'],
    'featured_price' => ['Type' => 'text', 'Default' => '29', 'Description' => 'Featured listing price']
]];}

function whmcs_domain_marketplace_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_marketplace` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain` VARCHAR(255) NOT NULL, `user_id` INT(11) NOT NULL, `price` DECIMAL(10,2) NOT NULL, `make_offer` TINYINT(1) DEFAULT 1, `description` TEXT, `category` VARCHAR(100), `is_featured` TINYINT(1) DEFAULT 0, `views` INT(11) DEFAULT 0, `status` ENUM('pending','active','sold','expired') DEFAULT 'pending', `expires_at` DATE, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain` (`domain`), KEY `status` (`status`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_offers` (`id` INT(11) NOT NULL AUTO_INCREMENT, `listing_id` INT(11) NOT NULL, `user_id` INT(11) NOT NULL, `offer_amount` DECIMAL(10,2) NOT NULL, `status` ENUM('pending','accepted','rejected','expired') DEFAULT 'pending', `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `listing_id` (`listing_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_marketplace_deactivate() { return ['status' => 'success']; }

function whmcs_domain_marketplace_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Marketplace</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as listings, SUM(is_featured=1) as featured, SUM(status='active') as active FROM mod_domain_marketplace"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Total Listings: ' . $stats['listings'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-warning"><div class="panel-body"><p>Featured: ' . $stats['featured'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Active: ' . $stats['active'] . '</p></div></div></div></div>';
    
    if ($_POST['create_listing']) {
        $domain = db_escape_string($_POST['domain']);
        $price = (float)$_POST['price'];
        $description = db_escape_string($_POST['description']);
        $category = db_escape_string($_POST['category']);
        $featured = isset($_POST['is_featured']) ? 1 : 0;
        insert_query('mod_domain_marketplace', ['domain' => $domain, 'price' => $price, 'description' => $description, 'category' => $category, 'is_featured' => $featured, 'status' => 'active', 'expires_at' => date('Y-m-d', strtotime('+30 days'))]);
        echo '<div class="alert alert-success">Listing created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Listing</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><input type="text" name="domain" class="form-control" placeholder="example.com"></div>
          <div class="form-group"><label>Price</label><input type="number" step="0.01" name="price" class="form-control"></div>
          <div class="form-group"><label>Category</label><input type="text" name="category" class="form-control" placeholder="e.g., Brandable, Tech, Finance"></div>
          <div class="form-group"><label>Description</label><textarea name="description" class="form-control" rows="3"></textarea></div>
          <div class="checkbox"><label><input type="checkbox" name="is_featured"> Featured Listing</label></div>
          <button type="submit" name="create_listing" class="btn btn-primary">Create Listing</button></div></form>';
    
    $listings = select_query('mod_domain_marketplace', '*', '', 'is_featured DESC, created_at DESC', '', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Price</th><th>Category</th><th>Featured</th><th>Views</th><th>Status</th></tr></thead><tbody>';
    while ($l = mysql_fetch_array($listings)) { 
        echo '<tr><td>' . $l['domain'] . '</td><td>$' . number_format($l['price'], 2) . '</td><td>' . $l['category'] . '</td><td>' . ($l['is_featured'] ? '<span class="label label-warning">Yes</span>' : 'No') . '</td><td>' . $l['views'] . '</td><td>' . ucfirst($l['status']) . '</td></tr>'; 
    }
    echo '</tbody></table></div>';
}

function makeOffer($listingId, $userId, $amount) {
    insert_query('mod_domain_offers', ['listing_id' => $listingId, 'user_id' => $userId, 'offer_amount' => $amount]);
    return ['success' => true, 'message' => 'Offer submitted'];
}

function acceptOffer($offerId) {
    update_query('mod_domain_offers', ['status' => 'accepted'], ['id' => $offerId]);
    $offer = mysql_fetch_array(select_query('mod_domain_offers', 'listing_id', ['id' => $offerId]));
    if ($offer) {
        update_query('mod_domain_marketplace', ['status' => 'sold'], ['id' => $offer['listing_id']]);
    }
    return ['success' => true, 'message' => 'Offer accepted'];
}
```