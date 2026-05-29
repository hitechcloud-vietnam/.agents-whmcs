# WHMCS Domain Transfer - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_transfer_config() { return ['name' => 'Domain Transfer', 'description' => 'Domain transfer management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_verify' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-verify transfer status']
]];}

function whmcs_domain_transfer_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_transfers` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain` VARCHAR(255) NOT NULL, `epp_code` VARCHAR(100), `status` ENUM('pending','initiated','auth_ok','completed','rejected','cancelled') DEFAULT 'pending', `initiated_at` DATETIME DEFAULT CURRENT_TIMESTAMP, `completed_at` DATETIME DEFAULT NULL, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_transfer_deactivate() { return ['status' => 'success']; }

function whmcs_domain_transfer_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Transfers</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as pending FROM mod_domain_transfers WHERE status IN ('pending','initiated','auth_ok')"));
    echo '<div class="panel panel-warning"><div class="panel-body"><p>Pending Transfers: ' . $stats['pending'] . '</p></div></div>';
    
    $transfers = select_query('mod_domain_transfers', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable"><thead><tr><th>Domain</th><th>EPP</th><th>Status</th><th>Initiated</th><th>Completed</th></tr></thead><tbody>';
    while ($t = mysql_fetch_array($transfers)) { echo '<tr><td>' . $t['domain'] . '</td><td>' . ($t['epp_code'] ? 'Yes' : 'No') . '</td><td>' . ucfirst(str_replace('_', ' ', $t['status'])) . '</td><td>' . $t['initiated_at'] . '</td><td>' . ($t['completed_at'] ?: '-') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function initiateTransfer($domain, $eppCode) {
    insert_query('mod_domain_transfers', ['domain' => $domain, 'epp_code' => $eppCode, 'status' => 'initiated']);
    $result = localAPI('TransferDomain', ['domain' => $domain, 'eppcode' => $eppCode]);
    return $result;
}

add_hook('DomainTransferComplete', 1, function($vars) {
    update_query('mod_domain_transfers', ['status' => 'completed', 'completed_at' => date('Y-m-d H:i:s')], ['domain' => $vars['domain']]);
});
```