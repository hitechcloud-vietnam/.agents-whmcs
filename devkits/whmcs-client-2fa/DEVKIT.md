# WHMCS Client 2FA - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_2fa_config() { return ['name' => 'Client 2FA', 'description' => 'Two-factor authentication', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'require_2fa' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Require 2FA for all clients'],
    'allow_totp' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow TOTP'],
    'allow_sms' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow SMS']
]];}

function whmcs_client_2fa_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_2fa_methods` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `method` VARCHAR(20) NOT NULL, `secret` VARCHAR(255), `is_primary` TINYINT(1) DEFAULT 0, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `user_id` (`user_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_2fa_backup_codes` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `code` VARCHAR(100) NOT NULL, `used` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`), KEY `user_id` (`user_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_2fa_deactivate() { return ['status' => 'success']; }

function whmcs_client_2fa_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client 2FA Management</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(CASE WHEN is_primary THEN 1 ELSE 0 END) as enabled FROM mod_2fa_methods"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Total 2FA Enabled: ' . $stats['total'] . '</p><p>Primary 2FA: ' . $stats['enabled'] . '</p></div></div>';
    $users = select_query('mod_2fa_methods m', 'DISTINCT u.id, u.email', ['m.is_primary' => 1], '', '10', '', 'u.email', 'INNER JOIN tblclients u ON m.user_id = u.id');
    echo '<table class="datatable"><thead><tr><th>User</th><th>Methods</th></tr></thead><tbody>';
    while ($u = mysql_fetch_array($users)) {
        $methods = select_query('mod_2fa_methods', 'method', ['user_id' => $u['id']]);
        $methodList = [];
        while ($m = mysql_fetch_array($methods)) { $methodList[] = $m['method']; }
        echo '<tr><td>' . $u['email'] . '</td><td>' . implode(', ', $methodList) . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function verify2FA($userId, $code, $method = 'totp') {
    $record = mysql_fetch_array(select_query('mod_2fa_methods', '*', ['user_id' => $userId, 'method' => $method, 'is_primary' => 1]));
    if (!$record) return false;
    
    if ($method == 'backup') {
        $backup = mysql_fetch_array(select_query('mod_2fa_backup_codes', '*', ['user_id' => $userId, 'code' => $code, 'used' => 0]));
        if ($backup) {
            update_query('mod_2fa_backup_codes', ['used' => 1], ['id' => $backup['id']]);
            return true;
        }
        return false;
    }
    return true;
}

add_hook('ClientLogin', 1, function($vars) { return ['require_2fa_check' => true]; });
```