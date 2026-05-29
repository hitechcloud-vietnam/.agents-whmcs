# WHMCS Client API Keys - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_api_keys_config() { return ['name' => 'Client API Keys', 'description' => 'API key management', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_client_api_keys_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_api_keys` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `key_name` VARCHAR(255) NOT NULL, `api_key` VARCHAR(64) NOT NULL UNIQUE, `scopes` TEXT, `last_used` DATETIME DEFAULT NULL, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `api_key` (`api_key`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_api_key_usage` (`id` INT(11) NOT NULL AUTO_INCREMENT, `key_id` INT(11) NOT NULL, `endpoint` VARCHAR(255), `ip_address` VARCHAR(45), `used_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `key_id` (`key_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_api_keys_deactivate() { return ['status' => 'success']; }

function whmcs_client_api_keys_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client API Keys</h2>';
    if ($_POST['create_key']) {
        $apiKey = bin2hex(random_bytes(32));
        insert_query('mod_api_keys', ['user_id' => $_POST['user_id'], 'key_name' => $_POST['key_name'], 'api_key' => $apiKey, 'scopes' => json_encode($_POST['scopes'] ?? [])]);
        echo '<div class="alert alert-success">API Key created: ' . $apiKey . '</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create API Key</div><div class="panel-body">
          <div class="form-group"><label>User</label><select name="user_id" class="form-control">';
    $users = select_query('tblclients', 'id, email', '', 'email');
    while ($u = mysql_fetch_array($users)) { echo '<option value="' . $u['id'] . '">' . $u['email'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Key Name</label><input type="text" name="key_name" class="form-control" required /></div>
          <div class="form-group"><label>Scopes</label><br/>
          <label><input type="checkbox" name="scopes[]" value="read" /> Read</label>
          <label><input type="checkbox" name="scopes[]" value="write" /> Write</label>
          <label><input type="checkbox" name="scopes[]" value="billing" /> Billing</label></div>
          <button type="submit" name="create_key" class="btn btn-primary">Create Key</button></div></form>';
    $keys = select_query('mod_api_keys', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>User</th><th>Name</th><th>Key</th><th>Last Used</th><th>Created</th></tr></thead><tbody>';
    while ($k = mysql_fetch_array($keys)) {
        $u = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $k['user_id']]));
        echo '<tr><td>' . $u['email'] . '</td><td>' . $k['key_name'] . '</td><td>' . substr($k['api_key'], 0, 8) . '...</td><td>' . ($k['last_used'] ?: 'Never') . '</td><td>' . $k['created_at'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function validateApiKey($key) {
    $record = mysql_fetch_array(select_query('mod_api_keys', '*', ['api_key' => $key]));
    if ($record) {
        update_query('mod_api_keys', ['last_used' => date('Y-m-d H:i:s')], ['id' => $record['id']]);
        insert_query('mod_api_key_usage', ['key_id' => $record['id'], 'ip_address' => $_SERVER['REMOTE_ADDR']]);
        return $record;
    }
    return null;
}
```