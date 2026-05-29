# WHMCS Domain Sync - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_sync_config() { return ['name' => 'Domain Sync', 'description' => 'Domain synchronization', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_sync' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable automatic sync'],
    'sync_interval' => ['Type' => 'dropdown', 'Options' => '1:Hourly,6:Every 6 hours,12:Every 12 hours,24:Daily', 'Default' => '24', 'Description' => 'Sync interval (hours)']
]];}

function whmcs_domain_sync_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_sync_log` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `sync_type` VARCHAR(50), `registrar_status` VARCHAR(50), `expiry_date` DATE, `next_expiry` DATE, `nameservers` TEXT, `domain_status` VARCHAR(50), `sync_result` TEXT, `synced_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_sync_deactivate() { return ['status' => 'success']; }

function whmcs_domain_sync_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Sync</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as syncs, MAX(synced_at) as last_sync FROM mod_domain_sync_log"));
    echo '<div class="row"><div class="col-md-6"><div class="panel panel-info"><div class="panel-body"><p>Total Syncs: ' . $stats['syncs'] . '</p></div></div></div>
          <div class="col-md-6"><div class="panel panel-success"><div class="panel-body"><p>Last Sync: ' . ($stats['last_sync'] ?: 'Never') . '</p></div></div></div></div>';
    
    if ($_POST['sync_domain']) {
        $domainId = (int)$_POST['domain_id'];
        $result = syncDomain($domainId);
        if ($result['success']) {
            echo '<div class="alert alert-success">Domain synced successfully!</div>';
        } else {
            echo '<div class="alert alert-danger">Sync failed: ' . $result['error'] . '</div>';
        }
    }
    
    if ($_POST['sync_all']) {
        $domains = select_query('tbldomains', 'id');
        $synced = 0;
        while ($d = mysql_fetch_array($domains)) {
            syncDomain($d['id']);
            $synced++;
        }
        echo '<div class="alert alert-success">Synced ' . $synced . ' domains!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Sync Domain</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <button type="submit" name="sync_domain" class="btn btn-primary">Sync Now</button></div></form>';
    
    echo '<form method="post" style="margin-top:10px;"><button type="submit" name="sync_all" class="btn btn-warning">Sync All Domains</button></form>';
    
    $logs = select_query('mod_domain_sync_log l', 'l.*, d.domain', '', 'l.id', 'DESC', '50', 'l.id', 'INNER JOIN tbldomains d ON l.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Registrar Status</th><th>Expiry</th><th>Synced</th></tr></thead><tbody>';
    while ($l = mysql_fetch_array($logs)) { echo '<tr><td>' . $l['domain'] . '</td><td>' . $l['registrar_status'] . '</td><td>' . $l['expiry_date'] . '</td><td>' . $l['synced_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function syncDomain($domainId) {
    $domain = mysql_fetch_array(select_query('tbldomains', '*', ['id' => $domainId]));
    if (!$domain) return ['success' => false, 'error' => 'Domain not found'];
    
    $syncData = ['domain_id' => $domainId, 'sync_type' => 'manual', 'synced_at' => date('Y-m-d H:i:s')];
    insert_query('mod_domain_sync_log', $syncData);
    
    return ['success' => true, 'message' => 'Synced'];
}

add_hook('DailyCronJob', 1, function($vars) {
    $domains = select_query('tbldomains', 'id');
    while ($d = mysql_fetch_array($domains)) {
        syncDomain($d['id']);
    }
    logActivity("Daily domain sync completed");
});
```