# WHMCS Client Archive - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_archive_config() {
    return ['name' => 'Client Archive', 'description' => 'Archive inactive clients', 'author' => 'DevKit Generator', 'version' => '1.0.0'];
}

function whmcs_client_archive_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_archives` (
        `id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `archive_data` LONGTEXT, `archived_by` INT(11) DEFAULT NULL, `archived_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_archive_deactivate() { return ['status' => 'success']; }

function whmcs_client_archive_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Archive</h2>';
    
    if ($_POST['archive_client']) {
        $userId = (int)$_POST['user_id'];
        $client = mysql_fetch_array(select_query('tblclients', '*', ['id' => $userId]));
        
        insert_query('mod_client_archives', [
            'user_id' => $userId,
            'archive_data' => json_encode($client),
            'archived_by' => $_SESSION['adminid']
        ]);
        
        update_query('tblclients', ['status' => 'Archived'], ['id' => $userId]);
        echo '<div class="alert alert-success">Client archived!</div>';
    }
    
    if ($_POST['restore_client']) {
        $userId = (int)$_POST['archive_id'];
        $archive = mysql_fetch_array(select_query('mod_client_archives', '*', ['user_id' => $userId]));
        $data = json_decode($archive['archive_data'], true);
        
        update_query('tblclients', ['status' => 'Active'], ['id' => $userId]);
        delete_query('mod_client_archives', ['user_id' => $userId]);
        echo '<div class="alert alert-success">Client restored!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Archive Client</div>
          <div class="panel-body">
          <div class="form-group"><label>Select Client</label><select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname) as name', ['status' => 'Inactive'], 'firstname');
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div>
          <button type="submit" name="archive_client" class="btn btn-primary">Archive</button>
          </div></form>';
    
    $archives = select_query('mod_client_archives', '*', '', 'archived_at', 'DESC');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Client</th><th>Archived</th><th>Actions</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($archives)) {
        $c = mysql_fetch_array(select_query('tblclients', 'firstname, lastname', ['id' => $a['user_id']]));
        echo '<tr><td>' . $c['firstname'] . ' ' . $c['lastname'] . '</td><td>' . $a['archived_at'] . '</td>
              <td><form method="post" style="display:inline;"><input type="hidden" name="archive_id" value="' . $a['user_id'] . '" /><button type="submit" name="restore_client" class="btn btn-xs btn-success">Restore</button></form></td></tr>';
    }
    echo '</tbody></table></div>';
}
```