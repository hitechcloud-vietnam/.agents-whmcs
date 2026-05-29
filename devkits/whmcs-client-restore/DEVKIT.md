# WHMCS Client Restore - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_restore_config() { return ['name' => 'Client Restore', 'description' => 'Restore archived clients', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_client_restore_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_restore_log` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `restored_by` INT(11) DEFAULT NULL, `restored_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_restore_deactivate() { return ['status' => 'success']; }

function whmcs_client_restore_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Restore</h2>';
    if ($_POST['restore']) {
        $userId = (int)$_POST['user_id'];
        update_query('tblclients', ['status' => 'Active'], ['id' => $userId]);
        insert_query('mod_restore_log', ['user_id' => $userId, 'restored_by' => $_SESSION['adminid']]);
        echo '<div class="alert alert-success">Client restored!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Restore Client</div><div class="panel-body"><div class="form-group"><label>Client</label><select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname) as name', "status IN ('Inactive','Archived')", 'firstname');
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div><button type="submit" name="restore" class="btn btn-primary">Restore</button></div></form></div>';
}
```