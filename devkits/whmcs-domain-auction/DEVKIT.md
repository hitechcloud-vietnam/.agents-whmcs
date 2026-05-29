# WHMCS Domain Auction - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_auction_config() { return ['name' => 'Domain Auction', 'description' => 'Domain auction system', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auction_fee' => ['Type' => 'text', 'Default' => '5', 'Description' => 'Auction listing fee'],
    'buyer_premium' => ['Type' => 'text', 'Default' => '10', 'Description' => 'Buyer premium %'],
    'min_increment' => ['Type' => 'text', 'Default' => '1', 'Description' => 'Minimum bid increment']
]];}

function whmcs_domain_auction_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_auctions` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain` VARCHAR(255) NOT NULL, `user_id` INT(11) NOT NULL, `starting_price` DECIMAL(10,2), `reserve_price` DECIMAL(10,2), `buy_now_price` DECIMAL(10,2), `current_bid` DECIMAL(10,2) DEFAULT 0, `highest_bidder` INT(11), `end_time` DATETIME, `status` ENUM('scheduled','active','ended','sold','cancelled') DEFAULT 'scheduled', `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain` (`domain`), KEY `status` (`status`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_bids` (`id` INT(11) NOT NULL AUTO_INCREMENT, `auction_id` INT(11) NOT NULL, `user_id` INT(11) NOT NULL, `bid_amount` DECIMAL(10,2) NOT NULL, `is_winning` TINYINT(1) DEFAULT 0, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `auction_id` (`auction_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_auction_deactivate() { return ['status' => 'success']; }

function whmcs_domain_auction_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Auctions</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(status='active') as active, SUM(status='ended') as ended FROM mod_domain_auctions"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Total Auctions: ' . $stats['total'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Active: ' . $stats['active'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-warning"><div class="panel-body"><p>Ended: ' . $stats['ended'] . '</p></div></div></div></div>';
    
    if ($_POST['create_auction']) {
        $domain = db_escape_string($_POST['domain']);
        $startPrice = (float)$_POST['starting_price'];
        $endTime = db_escape_string($_POST['end_time']);
        insert_query('mod_domain_auctions', ['domain' => $domain, 'starting_price' => $startPrice, 'current_bid' => $startPrice, 'end_time' => $endTime, 'status' => 'active']);
        echo '<div class="alert alert-success">Auction created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Auction</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><input type="text" name="domain" class="form-control" placeholder="example.com"></div>
          <div class="form-group"><label>Starting Price</label><input type="number" step="0.01" name="starting_price" class="form-control"></div>
          <div class="form-group"><label>End Time</label><input type="datetime-local" name="end_time" class="form-control"></div>
          <button type="submit" name="create_auction" class="btn btn-primary">Create Auction</button></div></form>';
    
    $auctions = select_query('mod_domain_auctions', '*', "status IN ('active','ended')", 'end_time', 'ASC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Current Bid</th><th>Start Price</th><th>End Time</th><th>Status</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($auctions)) { 
        $statusClass = $a['status'] === 'active' ? 'success' : 'default';
        echo '<tr><td>' . $a['domain'] . '</td><td>$' . number_format($a['current_bid'], 2) . '</td><td>$' . number_format($a['starting_price'], 2) . '</td><td>' . $a['end_time'] . '</td><td><span class="label label-' . $statusClass . '">' . ucfirst($a['status']) . '</span></td></tr>'; 
    }
    echo '</tbody></table></div>';
}

function placeBid($auctionId, $userId, $amount) {
    $auction = mysql_fetch_array(select_query('mod_domain_auctions', '*', ['id' => $auctionId, 'status' => 'active']));
    if (!$auction || strtotime($auction['end_time']) < time()) return ['success' => false, 'error' => 'Auction not available'];
    
    $minBid = $auction['current_bid'] + 1;
    if ($amount < $minBid) return ['success' => false, 'error' => 'Bid must be at least $' . $minBid];
    
    update_query('mod_domain_bids', ['is_winning' => 0], ['auction_id' => $auctionId]);
    insert_query('mod_domain_bids', ['auction_id' => $auctionId, 'user_id' => $userId, 'bid_amount' => $amount, 'is_winning' => 1]);
    update_query('mod_domain_auctions', ['current_bid' => $amount, 'highest_bidder' => $userId], ['id' => $auctionId]);
    
    return ['success' => true, 'message' => 'Bid placed successfully'];
}
```