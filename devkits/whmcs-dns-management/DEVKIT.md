# WHMCS DNS Management - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_dns_management_config() { return ['name' => 'DNS Management', 'description' => 'Advanced DNS zone management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'soa_refresh' => ['Type' => 'text', 'Default' => '7200', 'Description' => 'SOA Refresh (seconds)'],
    'soa_retry' => ['Type' => 'text', 'Default' => '3600', 'Description' => 'SOA Retry (seconds)'],
    'soa_expire' => ['Type' => 'text', 'Default' => '1209600', 'Description' => 'SOA Expire (seconds)']
]];}

function whmcs_dns_management_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_dns_zones` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `zone_data` TEXT, `soa_email` VARCHAR(255), `last_sync` DATETIME, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_dns_records` (`id` INT(11) NOT NULL AUTO_INCREMENT, `zone_id` INT(11) NOT NULL, `name` VARCHAR(255), `type` VARCHAR(10), `value` TEXT, `ttl` INT(11) DEFAULT 3600, `priority` INT(11) DEFAULT 0, `disabled` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`), KEY `zone_id` (`zone_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_dns_management_deactivate() { return ['status' => 'success']; }

function whmcs_dns_management_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>DNS Zone Management</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as zones FROM mod_dns_zones"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Active Zones: ' . $stats['zones'] . '</p></div></div>';
    
    if ($_POST['add_zone']) {
        $domainId = (int)$_POST['domain_id'];
        insert_query('mod_dns_zones', ['domain_id' => $domainId, 'last_sync' => date('Y-m-d H:i:s')]);
        echo '<div class="alert alert-success">DNS zone created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create DNS Zone</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <button type="submit" name="add_zone" class="btn btn-primary">Create Zone</button></div></form>';
    
    $zones = select_query('mod_dns_zones z', 'z.*, d.domain', '', 'z.id', 'DESC', '50', 'z.id', 'INNER JOIN tbldomains d ON z.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Last Sync</th><th>Actions</th></tr></thead><tbody>';
    while ($z = mysql_fetch_array($zones)) { echo '<tr><td>' . $z['domain'] . '</td><td>' . ($z['last_sync'] ?: 'Never') . '</td><td><a href="?action=manage&id=' . $z['id'] . '" class="btn btn-xs btn-default">Manage</a></td></tr>'; }
    echo '</tbody></table></div>';
}
```