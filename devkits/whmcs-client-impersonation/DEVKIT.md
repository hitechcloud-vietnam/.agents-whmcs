# WHMCS Client Impersonation - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_impersonation_config() { return ['name' => 'Client Impersonation', 'description' => 'Staff impersonation', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'require_approval' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Require approval for impersonation'],
    'log_all_actions' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Log all actions during impersonation']
]];}

function whmcs_client_impersonation_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_impersonation_log` (`id` INT(11) NOT NULL AUTO_INCREMENT, `staff_id` INT(11) NOT NULL, `user_id` INT(11) NOT NULL, `action` VARCHAR(50) DEFAULT 'start', `details` TEXT, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_impersonation_requests` (`id` INT(11) NOT NULL AUTO_INCREMENT, `staff_id` INT(11) NOT NULL, `user_id` INT(11) NOT NULL, `reason` TEXT, `status` ENUM('pending','approved','rejected') DEFAULT 'pending', `reviewed_by` INT(11) DEFAULT NULL, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_impersonation_deactivate() { return ['status' => 'success']; }

function whmcs_client_impersonation_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Impersonation</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, COUNT(DISTINCT user_id) as clients FROM mod_impersonation_log"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Total Impersonations: ' . $stats['total'] . '</p><p>Unique Clients: ' . $stats['clients'] . '</p></div></div>';
    
    if ($_POST['request_impersonation']) {
        insert_query('mod_impersonation_requests', ['staff_id' => $_SESSION['adminid'], 'user_id' => $_POST['user_id'], 'reason' => $_POST['reason']]);
        echo '<div class="alert alert-success">Impersonation request submitted!</div>';
    }
    
    $log = select_query('mod_impersonation_log', '*', '', 'id', 'DESC', '20');
    echo '<table class="datatable"><thead><tr><th>Staff</th><th>Client</th><th>Action</th><th>Date</th></tr></thead><tbody>';
    while ($l = mysql_fetch_array($log)) {
        $s = mysql_fetch_array(select_query('tbladmin', 'username', ['id' => $l['staff_id']]));
        $c = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $l['user_id']]));
        echo '<tr><td>' . $s['username'] . '</td><td>' . $c['email'] . '</td><td>' . $l['action'] . '</td><td>' . $l['created_at'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function startImpersonation($staffId, $userId, $reason = '') {
    insert_query('mod_impersonation_log', ['staff_id' => $staffId, 'user_id' => $userId, 'action' => 'start', 'details' => $reason]);
    $_SESSION['impersonating'] = $userId;
    $_SESSION['original_admin'] = $staffId;
}

function endImpersonation($staffId) {
    insert_query('mod_impersonation_log', ['staff_id' => $staffId, 'user_id' => $_SESSION['impersonating'], 'action' => 'end']);
    unset($_SESSION['impersonating']);
}

add_hook('AdminAreaPage', 1, function($vars) {
    if (isset($_SESSION['impersonating'])) {
        return ['impersonating_user' => $_SESSION['impersonating']];
    }
});
```