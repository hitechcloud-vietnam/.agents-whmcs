# WHMCS Domain Lock - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_lock_config() { return ['name' => 'Domain Lock', 'description' => 'Domain registrar lock management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_lock' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-enable lock on new domains'],
    'unlock_request' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Allow unlock requests']
]];}

function whmcs_domain_lock_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_lock` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `lock_status` ENUM('locked','unlocked','pending') DEFAULT 'locked', `unlock_requested` TINYINT(1) DEFAULT 0, `unlock_reason` TEXT, `locked_at` DATETIME DEFAULT CURRENT_TIMESTAMP, `unlocked_at` DATETIME DEFAULT NULL, PRIMARY KEY (`id`), UNIQUE KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_lock_deactivate() { return ['status' => 'success']; }

function whmcs_domain_lock_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Lock</h2>';
    $stats = mysql_fetch_array(full_query("SELECT SUM(lock_status='locked') as locked, SUM(lock_status='unlocked') as unlocked, SUM(unlock_requested=1) as requests FROM mod_domain_lock"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Locked: ' . $stats['locked'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-warning"><div class="panel-body"><p>Unlocked: ' . $stats['unlocked'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Unlock Requests: ' . $stats['requests'] . '</p></div></div></div></div>';
    
    if ($_POST['toggle_lock']) {
        $domainId = (int)$_POST['domain_id'];
        $newStatus = db_escape_string($_POST['new_status']);
        update_query('mod_domain_lock', ['lock_status' => $newStatus, 'unlocked_at' => ($newStatus === 'unlocked' ? date('Y-m-d H:i:s') : NULL)], ['domain_id' => $domainId]);
        echo '<div class="alert alert-success">Lock status updated!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Toggle Lock Status</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Status</label><select name="new_status" class="form-control"><option value="locked">Locked</option><option value="unlocked">Unlocked</option></select></div>
          <button type="submit" name="toggle_lock" class="btn btn-primary">Update</button></div></form>';
    
    $locks = select_query('mod_domain_lock l', 'l.*, d.domain', '', 'l.id', 'DESC', '50', 'l.id', 'INNER JOIN tbldomains d ON l.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Status</th><th>Unlock Request</th><th>Locked At</th></tr></thead><tbody>';
    while ($l = mysql_fetch_array($locks)) { echo '<tr><td>' . $l['domain'] . '</td><td><span class="label label-' . ($l['lock_status'] === 'locked' ? 'success' : 'warning') . '">' . ucfirst($l['lock_status']) . '</span></td><td>' . ($l['unlock_requested'] ? 'Yes' : 'No') . '</td><td>' . $l['locked_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function requestUnlock($domainId, $reason) {
    update_query('mod_domain_lock', ['unlock_requested' => 1, 'unlock_reason' => $reason], ['domain_id' => $domainId]);
}

add_hook('AfterDomainCreate', 1, function($vars) {
    $config = ModuleBuildParams('whmcs_domain_lock');
    if (!empty($config['auto_lock'])) {
        insert_query('mod_domain_lock', ['domain_id' => $vars['domainid'], 'lock_status' => 'locked', 'locked_at' => date('Y-m-d H:i:s')]);
    }
});
```