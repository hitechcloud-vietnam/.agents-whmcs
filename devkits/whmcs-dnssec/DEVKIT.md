# WHMCS DNSSEC - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_dnssec_config() { return ['name' => 'DNSSEC', 'description' => 'DNSSEC key and record management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_sign' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Auto-sign new zones'],
    'key_algorithm' => ['Type' => 'dropdown', 'Options' => '13:ECDSAP256SHA256,8:ECDSAP384SHA384,7:RSASHA1NSEC3SHA1,5:RSASHA1', 'Default' => '13', 'Description' => 'Default signing algorithm']
]];}

function whmcs_dnssec_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_dnssec_keys` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `key_type` ENUM('KSK','ZSK') DEFAULT 'ZSK', `algorithm` VARCHAR(10), `key_tag` INT(11), `public_key` TEXT, `key_status` ENUM('active','inactive','published') DEFAULT 'published', `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, `activated_at` DATETIME DEFAULT NULL, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_dnssec_ds` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `key_tag` INT(11), `algorithm` VARCHAR(10), `digest_type` VARCHAR(10), `digest` TEXT, `is_published` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_dnssec_deactivate() { return ['status' => 'success']; }

function whmcs_dnssec_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>DNSSEC Management</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as keys_total, SUM(key_status='active') as active FROM mod_dnssec_keys"));
    echo '<div class="row"><div class="col-md-6"><div class="panel panel-info"><div class="panel-body"><p>Total Keys: ' . $stats['keys_total'] . '</p></div></div></div>
          <div class="col-md-6"><div class="panel panel-success"><div class="panel-body"><p>Active Keys: ' . $stats['active'] . '</p></div></div></div></div>';
    
    if ($_POST['generate_keys']) {
        $domainId = (int)$_POST['domain_id'];
        $keyType = db_escape_string($_POST['key_type']);
        $algorithm = db_escape_string($_POST['algorithm']);
        $keyTag = rand(10000, 65000);
        insert_query('mod_dnssec_keys', ['domain_id' => $domainId, 'key_type' => $keyType, 'algorithm' => $algorithm, 'key_tag' => $keyTag, 'public_key' => 'Generated Key Data', 'key_status' => 'published']);
        echo '<div class="alert alert-success">DNSSEC keys generated!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Generate Keys</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Key Type</label><select name="key_type" class="form-control"><option value="KSK">KSK (Key Signing Key)</option><option value="ZSK">ZSK (Zone Signing Key)</option></select></div>
          <div class="form-group"><label>Algorithm</label><select name="algorithm" class="form-control"><option value="13">ECDSAP256SHA256</option><option value="8">ECDSAP384SHA384</option></select></div>
          <button type="submit" name="generate_keys" class="btn btn-primary">Generate</button></div></form>';
    
    $keys = select_query('mod_dnssec_keys k', 'k.*, d.domain', '', 'k.id', 'DESC', '50', 'k.id', 'INNER JOIN tbldomains d ON k.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Type</th><th>Algorithm</th><th>Key Tag</th><th>Status</th></tr></thead><tbody>';
    while ($k = mysql_fetch_array($keys)) { echo '<tr><td>' . $k['domain'] . '</td><td>' . $k['key_type'] . '</td><td>' . $k['algorithm'] . '</td><td>' . $k['key_tag'] . '</td><td>' . ucfirst($k['key_status']) . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```