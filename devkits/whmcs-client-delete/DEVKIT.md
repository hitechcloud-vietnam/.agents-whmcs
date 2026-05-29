# WHMCS Client Delete - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_delete_config() { return ['name' => 'Client Delete', 'description' => 'Safe client deletion', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_client_delete_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_delete_backup` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `backup_data` LONGTEXT, `deleted_by` INT(11) DEFAULT NULL, `deleted_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_delete_deactivate() { return ['status' => 'success']; }

function whmcs_client_delete_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Delete</h2>';
    if ($_POST['delete_client'] && $_POST['confirm'] == 'DELETE') {
        $userId = (int)$_POST['user_id'];
        $client = mysql_fetch_array(select_query('tblclients', '*', ['id' => $userId]));
        insert_query('mod_delete_backup', ['user_id' => $userId, 'backup_data' => json_encode($client), 'deleted_by' => $_SESSION['adminid']]);
        delete_query('tblclients', ['id' => $userId]);
        echo '<div class="alert alert-success">Client deleted (backup created)!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading" style="background:#dc3545;color:white;">Delete Client</div><div class="panel-body"><div class="form-group"><label>Client</label><select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname) as name', ['status' => 'Inactive'], 'firstname');
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div><div class="alert alert-warning"><strong>Warning:</strong> This will permanently delete the client. A backup will be created.</div>
    <div class="form-group"><label>Type DELETE to confirm</label><input type="text" name="confirm" class="form-control" /></div>
    <button type="submit" name="delete_client" class="btn btn-danger">Delete Client</button></div></form></div>';
}
```