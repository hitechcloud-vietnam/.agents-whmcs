# WHMCS Tiered Pricing - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_tiered_pricing_config() {
    return [
        'name' => 'Tiered Pricing',
        'description' => 'Multi-level pricing tiers with customer-based pricing',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'auto_upgrade' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-upgrade customer tiers'],
            'show_comparison' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show tier comparison']
        ]
    ];
}

function whmcs_tiered_pricing_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_tiered_pricing_tiers` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `tier_name` VARCHAR(255) NOT NULL,
        `tier_level` INT(11) NOT NULL,
        `discount_percent` DECIMAL(5,2) DEFAULT 0.00,
        `min_purchase` DECIMAL(10,2) DEFAULT 0.00,
        `benefits` TEXT,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_customer_tiers` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `tier_id` INT(11) NOT NULL,
        `total_spent` DECIMAL(10,2) DEFAULT 0.00,
        `assigned_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_tiered_pricing_deactivate() { return ['status' => 'success']; }

function whmcs_tiered_pricing_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Tiered Pricing</h2>';
    
    if ($_POST['save_tier']) {
        insert_query('mod_tiered_pricing_tiers', [
            'tier_name' => $_POST['tier_name'],
            'tier_level' => $_POST['tier_level'],
            'discount_percent' => $_POST['discount'],
            'min_purchase' => $_POST['min_purchase'],
            'benefits' => $_POST['benefits']
        ]);
        echo '<div class="alert alert-success">Tier created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Create Pricing Tier</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>Tier Name</label>
          <input type="text" name="tier_name" class="form-control" required /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Tier Level</label>
          <input type="number" name="tier_level" class="form-control" required /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Discount %</label>
          <input type="number" step="0.01" name="discount" class="form-control" /></div></div>
          </div>
          <div class="form-group"><label>Min Purchase Required</label>
          <input type="number" step="0.01" name="min_purchase" class="form-control" /></div>
          <div class="form-group"><label>Benefits</label>
          <textarea name="benefits" class="form-control" placeholder="One benefit per line"></textarea></div>
          <button type="submit" name="save_tier" class="btn btn-primary">Create Tier</button>
          </div></form>';
    
    $tiers = select_query("mod_tiered_pricing_tiers", "*", "", "tier_level");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Name</th><th>Level</th><th>Discount</th><th>Min Purchase</th></tr></thead>
          <tbody>';
    while ($t = mysql_fetch_array($tiers)) {
        echo '<tr><td>' . $t['tier_name'] . '</td>
              <td>' . $t['tier_level'] . '</td>
              <td>' . $t['discount_percent'] . '%</td>
              <td>$' . $t['min_purchase'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('CalculateItemPrice', 1, function($vars) {
    $uid = $_SESSION['uid'] ?? 0;
    $tier = mysql_fetch_array(select_query("mod_customer_tiers", "tier_id", ["user_id" => $uid]));
    if ($tier) {
        $tierInfo = mysql_fetch_array(select_query("mod_tiered_pricing_tiers", "discount_percent", ["id" => $tier['tier_id']]));
        if ($tierInfo) {
            return ['discount_percent' => $tierInfo['discount_percent']];
        }
    }
});
```