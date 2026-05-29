# WHMCS Domain DNS - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_dns_config() { return ['name' => 'Domain DNS', 'description' => 'Domain DNS management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'default_records' => ['Type' => 'textarea', 'Default' => '', 'Description' => 'Default DNS records (JSON)']
]];}

function whmcs_domain_dns_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_dns_records` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `name` VARCHAR(255), `type` VARCHAR(10) DEFAULT 'A', `value` VARCHAR(255), `ttl` INT(11) DEFAULT 3600, `priority` INT(11) DEFAULT 0, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_dns_deactivate() { return ['status' => 'success']; }

function whmcs_domain_dns_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain DNS</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total FROM mod_domain_dns_records"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Total DNS Records: ' . $stats['total'] . '</p></div></div>';
    
    if ($_POST['add_record']) {
        $domainId = (int)$_POST['domain_id'];
        $name = db_escape_string($_POST['name']);
        $type = db_escape_string($_POST['type']);
        $value = db_escape_string($_POST['value']);
        $ttl = (int)$_POST['ttl'];
        $priority = (int)$_POST['priority'];
        insert_query('mod_domain_dns_records', ['domain_id' => $domainId, 'name' => $name, 'type' => $type, 'value' => $value, 'ttl' => $ttl, 'priority' => $priority]);
        echo '<div class="alert alert-success">DNS record added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add DNS Record</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Name</label><input type="text" name="name" class="form-control" placeholder="@ or subdomain"></div>
          <div class="form-group"><label>Type</label><select name="type" class="form-control"><option value="A">A</option><option value="AAAA">AAAA</option><option value="CNAME">CNAME</option><option value="MX">MX</option><option value="TXT">TXT</option></select></div>
          <div class="form-group"><label>Value</label><input type="text" name="value" class="form-control"></div>
          <div class="row"><div class="col-md-6"><div class="form-group"><label>TTL</label><input type="number" name="ttl" value="3600" class="form-control"></div></div>
          <div class="col-md-6"><div class="form-group"><label>Priority</label><input type="number" name="priority" value="0" class="form-control"></div></div></div>
          <button type="submit" name="add_record" class="btn btn-primary">Add Record</button></div></form>';
    
    $records = select_query('mod_domain_dns_records r', 'r.*, d.domain', '', 'r.id', 'DESC', '50', 'r.id', 'INNER JOIN tbldomains d ON r.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Name</th><th>Type</th><th>Value</th><th>TTL</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($records)) { echo '<tr><td>' . $r['domain'] . '</td><td>' . $r['name'] . '</td><td>' . $r['type'] . '</td><td>' . $r['value'] . '</td><td>' . $r['ttl'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function getDomainDNS($domainId) {
    return select_query('mod_domain_dns_records', '*', ['domain_id' => $domainId]);
}

add_hook('AfterDomainCreate', 1, function($vars) {
    $config = ModuleBuildParams('whmcs_domain_dns');
    if (!empty($config['default_records'])) {
        $defaults = json_decode($config['default_records'], true);
        foreach ($defaults as $record) {
            insert_query('mod_domain_dns_records', array_merge($record, ['domain_id' => $vars['domainid']]));
        }
    }
});
```