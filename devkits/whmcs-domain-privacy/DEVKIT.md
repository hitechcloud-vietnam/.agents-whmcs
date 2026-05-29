# WHMCS Domain Privacy - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_privacy_config() { return ['name' => 'Domain Privacy', 'description' => 'Domain privacy protection', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_enable' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Auto-enable privacy'],
    'privacy_price' => ['Type' => 'text', 'Default' => '9.99', 'Description' => 'Annual privacy price']
]];}

function whmcs_domain_privacy_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_privacy` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `privacy_enabled` TINYINT(1) DEFAULT 0, `privacy_product_id` INT(11) DEFAULT NULL, `enabled_at` DATETIME DEFAULT NULL, PRIMARY KEY (`id`), UNIQUE KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_privacy_deactivate() { return ['status' => 'success']; }

function whmcs_domain_privacy_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Privacy</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as enabled FROM mod_domain_privacy WHERE privacy_enabled=1"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Domains with Privacy: ' . $stats['enabled'] . '</p></div></div>';
    
    if ($_POST['enable_privacy']) {
        $domainId = (int)$_POST['domain_id'];
        $exists = mysql_num_rows(select_query('mod_domain_privacy', 'id', ['domain_id' => $domainId]));
        if ($exists) {
            update_query('mod_domain_privacy', ['privacy_enabled' => 1, 'enabled_at' => date('Y-m-d H:i:s')], ['domain_id' => $domainId]);
        } else {
            insert_query('mod_domain_privacy', ['domain_id' => $domainId, 'privacy_enabled' => 1, 'enabled_at' => date('Y-m-d H:i:s')]);
        }
        echo '<div class="alert alert-success">Privacy enabled!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Enable Privacy</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <button type="submit" name="enable_privacy" class="btn btn-primary">Enable Privacy</button></div></form>';
    
    $privacy = select_query('mod_domain_privacy p', 'p.*, d.domain', '', 'p.id', 'DESC', '50', 'p.id', 'INNER JOIN tbldomains d ON p.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Enabled</th><th>Date</th></tr></thead><tbody>';
    while ($p = mysql_fetch_array($privacy)) { echo '<tr><td>' . $p['domain'] . '</td><td>' . ($p['privacy_enabled'] ? 'Yes' : 'No') . '</td><td>' . ($p['enabled_at'] ?: '-') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```