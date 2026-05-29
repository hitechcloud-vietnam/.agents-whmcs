# WHMCS Domain Pricing - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_pricing_config() { return ['name' => 'Domain Pricing', 'description' => 'Domain pricing management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'default_markup' => ['Type' => 'text', 'Default' => '10', 'Description' => 'Default markup %'],
    'bulk_discount' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable bulk discounts']
]];}

function whmcs_domain_pricing_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_pricing` (`id` INT(11) NOT NULL AUTO_INCREMENT, `tld` VARCHAR(50) NOT NULL, `register_price` DECIMAL(10,2), `renew_price` DECIMAL(10,2), `transfer_price` DECIMAL(10,2), `currency` VARCHAR(10) DEFAULT 'USD', `is_active` TINYINT(1) DEFAULT 1, `bulk_qty` INT(11) DEFAULT 5, `bulk_discount` DECIMAL(5,2) DEFAULT 0, `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `tld` (`tld`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_pricing_deactivate() { return ['status' => 'success']; }

function whmcs_domain_pricing_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Pricing</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as tlds, AVG(register_price) as avg_price FROM mod_domain_pricing"));
    echo '<div class="row"><div class="col-md-6"><div class="panel panel-info"><div class="panel-body"><p>TLDs Configured: ' . $stats['tlds'] . '</p></div></div></div>
          <div class="col-md-6"><div class="panel panel-success"><div class="panel-body"><p>Avg. Register Price: $' . number_format($stats['avg_price'], 2) . '</p></div></div></div></div>';
    
    if ($_POST['update_pricing']) {
        $tld = db_escape_string($_POST['tld']);
        $regPrice = (float)$_POST['register_price'];
        $renewPrice = (float)$_POST['renew_price'];
        $transferPrice = (float)$_POST['transfer_price'];
        $existing = mysql_fetch_array(select_query('mod_domain_pricing', 'id', ['tld' => $tld]));
        $data = ['register_price' => $regPrice, 'renew_price' => $renewPrice, 'transfer_price' => $transferPrice];
        if ($existing) {
            update_query('mod_domain_pricing', $data, ['tld' => $tld]);
        } else {
            insert_query('mod_domain_pricing', array_merge(['tld' => $tld], $data));
        }
        echo '<div class="alert alert-success">Pricing updated!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Update Pricing</div><div class="panel-body">
          <div class="form-group"><label>TLD</label><input type="text" name="tld" class="form-control" placeholder=".com, .net, etc"></div>
          <div class="row"><div class="col-md-4"><div class="form-group"><label>Register</label><input type="number" step="0.01" name="register_price" class="form-control"></div></div>
          <div class="col-md-4"><div class="form-group"><label>Renew</label><input type="number" step="0.01" name="renew_price" class="form-control"></div></div>
          <div class="col-md-4"><div class="form-group"><label>Transfer</label><input type="number" step="0.01" name="transfer_price" class="form-control"></div></div></div>
          <button type="submit" name="update_pricing" class="btn btn-primary">Update</button></div></form>';
    
    $pricing = select_query('mod_domain_pricing', '*', '', 'register_price', 'ASC', '100');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>TLD</th><th>Register</th><th>Renew</th><th>Transfer</th><th>Status</th></tr></thead><tbody>';
    while ($p = mysql_fetch_array($pricing)) { 
        echo '<tr><td>' . $p['tld'] . '</td><td>$' . number_format($p['register_price'], 2) . '</td><td>$' . number_format($p['renew_price'], 2) . '</td><td>$' . number_format($p['transfer_price'], 2) . '</td><td>' . ($p['is_active'] ? 'Active' : 'Inactive') . '</td></tr>'; 
    }
    echo '</tbody></table></div>';
}

function getDomainPrice($tld, $type = 'register') {
    $field = $type . '_price';
    $pricing = mysql_fetch_array(select_query('mod_domain_pricing', $field, ['tld' => $tld, 'is_active' => 1]));
    return $pricing ? $pricing[$field] : null;
}
```