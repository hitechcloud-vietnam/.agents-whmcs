# WHMCS Domain Redirect - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_redirect_config() { return ['name' => 'Domain Redirect', 'description' => 'URL forwarding and redirects', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'redirect_type' => ['Type' => 'dropdown', 'Options' => '301:Temporary (302),302:Permanent (301)', 'Default' => '302', 'Description' => 'Default redirect type']
]];}

function whmcs_domain_redirect_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_redirects` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `source_path` VARCHAR(255) DEFAULT '/', `target_url` VARCHAR(500) NOT NULL, `redirect_type` VARCHAR(10) DEFAULT '302', `is_wildcard` TINYINT(1) DEFAULT 0, `is_active` TINYINT(1) DEFAULT 1, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_redirect_deactivate() { return ['status' => 'success']; }

function whmcs_domain_redirect_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Redirects</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(is_active=1) as active FROM mod_domain_redirects"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Total Redirects: ' . $stats['total'] . ' | Active: ' . $stats['active'] . '</p></div></div>';
    
    if ($_POST['add_redirect']) {
        $domainId = (int)$_POST['domain_id'];
        $source = db_escape_string($_POST['source_path']);
        $target = db_escape_string($_POST['target_url']);
        $type = db_escape_string($_POST['redirect_type']);
        insert_query('mod_domain_redirects', ['domain_id' => $domainId, 'source_path' => $source, 'target_url' => $target, 'redirect_type' => $type]);
        echo '<div class="alert alert-success">Redirect added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Redirect</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Source Path</label><input type="text" name="source_path" value="/" class="form-control"></div>
          <div class="form-group"><label>Target URL</label><input type="url" name="target_url" class="form-control" placeholder="https://"></div>
          <div class="form-group"><label>Type</label><select name="redirect_type" class="form-control"><option value="301">301 - Permanent</option><option value="302">302 - Temporary</option></select></div>
          <button type="submit" name="add_redirect" class="btn btn-primary">Add Redirect</button></div></form>';
    
    $redirects = select_query('mod_domain_redirects r', 'r.*, d.domain', '', 'r.id', 'DESC', '50', 'r.id', 'INNER JOIN tbldomains d ON r.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Source</th><th>Target</th><th>Type</th><th>Active</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($redirects)) { echo '<tr><td>' . $r['domain'] . '</td><td>' . $r['source_path'] . '</td><td>' . $r['target_url'] . '</td><td>' . $r['redirect_type'] . '</td><td>' . ($r['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```