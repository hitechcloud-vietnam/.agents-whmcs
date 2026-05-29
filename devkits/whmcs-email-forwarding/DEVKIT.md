# WHMCS Email Forwarding - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_email_forwarding_config() { return ['name' => 'Email Forwarding', 'description' => 'Email forwarding rules', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'max_rules' => ['Type' => 'text', 'Default' => '50', 'Description' => 'Max forwarding rules per domain'],
    'enable_catchall' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable catch-all forwarding']
]];}

function whmcs_email_forwarding_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_email_forwarding` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `source_address` VARCHAR(255) NOT NULL, `destination_address` VARCHAR(255) NOT NULL, `is_catchall` TINYINT(1) DEFAULT 0, `is_active` TINYINT(1) DEFAULT 1, `forward_count` INT(11) DEFAULT 0, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_email_forwarding_deactivate() { return ['status' => 'success']; }

function whmcs_email_forwarding_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Email Forwarding</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(is_active=1) as active, SUM(is_catchall=1) as catchall FROM mod_email_forwarding"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Total Rules: ' . $stats['total'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Active: ' . $stats['active'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-warning"><div class="panel-body"><p>Catch-All: ' . $stats['catchall'] . '</p></div></div></div></div>';
    
    if ($_POST['add_forward']) {
        $domainId = (int)$_POST['domain_id'];
        $source = db_escape_string($_POST['source']);
        $dest = db_escape_string($_POST['destination']);
        $catchall = isset($_POST['is_catchall']) ? 1 : 0;
        insert_query('mod_email_forwarding', ['domain_id' => $domainId, 'source_address' => $source, 'destination_address' => $dest, 'is_catchall' => $catchall]);
        echo '<div class="alert alert-success">Forwarding rule added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Forwarding Rule</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Source Address</label><input type="text" name="source" class="form-control" placeholder="user or * for catch-all"></div>
          <div class="form-group"><label>Destination</label><input type="email" name="destination" class="form-control" placeholder="forward@example.com"></div>
          <div class="checkbox"><label><input type="checkbox" name="is_catchall"> Catch-All</label></div>
          <button type="submit" name="add_forward" class="btn btn-primary">Add Rule</button></div></form>';
    
    $forwards = select_query('mod_email_forwarding f', 'f.*, d.domain', '', 'f.id', 'DESC', '50', 'f.id', 'INNER JOIN tbldomains d ON f.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Source</th><th>Destination</th><th>Catch-All</th><th>Active</th></tr></thead><tbody>';
    while ($f = mysql_fetch_array($forwards)) { echo '<tr><td>' . $f['domain'] . '</td><td>' . $f['source_address'] . '</td><td>' . $f['destination_address'] . '</td><td>' . ($f['is_catchall'] ? 'Yes' : 'No') . '</td><td>' . ($f['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```