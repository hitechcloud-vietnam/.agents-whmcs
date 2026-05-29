# WHMCS Client Authentication - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_authentication_config() {
    return ['name' => 'Client Authentication', 'description' => 'Enhanced authentication', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
        'require_strong' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Require strong passwords'],
        'sso_enabled' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Enable SSO']
    ]];
}

function whmcs_client_authentication_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_auth_policies` (`id` INT(11) NOT NULL AUTO_INCREMENT, `policy_name` VARCHAR(255) NOT NULL, `min_length` INT(11) DEFAULT 8, `require_special` TINYINT(1) DEFAULT 0, `require_number` TINYINT(1) DEFAULT 1, `session_timeout` INT(11) DEFAULT 3600, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_sso_providers` (`id` INT(11) NOT NULL AUTO_INCREMENT, `provider_name` VARCHAR(100) NOT NULL, `client_id` VARCHAR(255), `client_secret` VARCHAR(255), `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_authentication_deactivate() { return ['status' => 'success']; }

function whmcs_client_authentication_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Authentication</h2>';
    if ($_POST['add_policy']) {
        insert_query('mod_auth_policies', ['policy_name' => $_POST['policy_name'], 'min_length' => $_POST['min_length'] ?: 8, 'require_special' => $_POST['require_special'] ?? 0]);
        echo '<div class="alert alert-success">Policy created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Auth Policy</div><div class="panel-body">
          <div class="form-group"><label>Policy Name</label><input type="text" name="policy_name" class="form-control" required /></div>
          <div class="form-group"><label>Min Password Length</label><input type="number" name="min_length" class="form-control" value="8" /></div>
          <div class="form-group"><label><input type="checkbox" name="require_special" value="1" /> Require Special Characters</label></div>
          <button type="submit" name="add_policy" class="btn btn-primary">Create Policy</button></div></form>';
    $policies = select_query('mod_auth_policies', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Min Length</th><th>Special</th><th>Active</th></tr></thead><tbody>';
    while ($p = mysql_fetch_array($policies)) { echo '<tr><td>' . $p['policy_name'] . '</td><td>' . $p['min_length'] . '</td><td>' . ($p['require_special'] ? 'Yes' : 'No') . '</td><td>' . ($p['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

add_hook('ClientLogin', 1, function($vars) { return ['auth_policy_check' => true]; });
```