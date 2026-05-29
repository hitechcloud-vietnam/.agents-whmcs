# WHMCS Client Sessions - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_sessions_config() { return ['name' => 'Client Sessions', 'description' => 'Session management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'max_sessions' => ['Type' => 'text', 'Default' => '3', 'Description' => 'Max concurrent sessions']
]];}

function whmcs_client_sessions_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_sessions` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `session_id` VARCHAR(128) NOT NULL, `ip_address` VARCHAR(45), `user_agent` VARCHAR(255), `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, `last_activity` DATETIME DEFAULT CURRENT_TIMESTAMP, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`), KEY `user_id` (`user_id`), KEY `session_id` (`session_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_sessions_deactivate() { return ['status' => 'success']; }

function whmcs_client_sessions_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Sessions</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as active FROM mod_client_sessions WHERE is_active=1"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Active Sessions: ' . $stats['active'] . '</p></div></div>';
    
    if ($_POST['logout_session']) {
        update_query('mod_client_sessions', ['is_active' => 0], ['id' => $_POST['session_id']]);
        echo '<div class="alert alert-success">Session terminated!</div>';
    }
    
    $sessions = select_query('mod_client_sessions', '*', ['is_active' => 1], 'last_activity', 'DESC', '50');
    echo '<table class="datatable"><thead><tr><th>User</th><th>IP</th><th>User Agent</th><th>Last Activity</th><th>Actions</th></tr></thead><tbody>';
    while ($s = mysql_fetch_array($sessions)) {
        $u = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $s['user_id']]));
        echo '<tr><td>' . $u['email'] . '</td><td>' . $s['ip_address'] . '</td><td>' . substr($s['user_agent'], 0, 30) . '</td><td>' . $s['last_activity'] . '</td>
              <td><form method="post" style="display:inline;"><input type="hidden" name="session_id" value="' . $s['id'] . '" /><button name="logout_session" class="btn btn-xs btn-danger">Terminate</button></form></td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('ClientLogin', 1, function($vars) {
    $maxSessions = get_config('max_sessions') ?: 3;
    $active = mysql_num_rows(select_query('mod_client_sessions', 'id', ['user_id' => $vars['userid'], 'is_active' => 1]));
    if ($active >= $maxSessions) {
        $oldest = mysql_fetch_array(select_query('mod_client_sessions', 'id', ['user_id' => $vars['userid'], 'is_active' => 1], 'last_activity', 'ASC', '1'));
        if ($oldest) update_query('mod_client_sessions', ['is_active' => 0], ['id' => $oldest['id']]);
    }
    insert_query('mod_client_sessions', ['user_id' => $vars['userid'], 'session_id' => session_id(), 'ip_address' => $_SERVER['REMOTE_ADDR'], 'user_agent' => $_SERVER['HTTP_USER_AGENT']]);
});
```