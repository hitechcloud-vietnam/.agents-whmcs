# WHMCS Client Merge - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_merge_config() {
    return ['name' => 'Client Merge', 'description' => 'Merge duplicate client accounts', 'author' => 'DevKit Generator', 'version' => '1.0.0'];
}

function whmcs_client_merge_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_merge_log` (
        `id` INT(11) NOT NULL AUTO_INCREMENT, `source_id` INT(11) NOT NULL, `target_id` INT(11) NOT NULL, `merged_fields` TEXT, `merged_services` INT(11) DEFAULT 0, `merged_invoices` INT(11) DEFAULT 0, `merged_tickets` INT(11) DEFAULT 0, `merged_by` INT(11) DEFAULT NULL, `merged_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_merge_deactivate() { return ['status' => 'success']; }

function whmcs_client_merge_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Merge</h2>';
    
    if ($_POST['merge_clients']) {
        $sourceId = (int)$_POST['source_id'];
        $targetId = (int)$_POST['target_id'];
        $keepTarget = $_POST['keep_target'] ?? true;
        
        // Transfer services
        $services = select_query('tblhosting', 'COUNT(*) as count', ['userid' => $sourceId]);
        $svcCount = mysql_fetch_array($services);
        update_query('tblhosting', ['userid' => $targetId], ['userid' => $sourceId]);
        
        // Transfer invoices
        $invoices = select_query('tblinvoices', 'COUNT(*) as count', ['userid' => $sourceId]);
        $invCount = mysql_fetch_array($invoices);
        update_query('tblinvoices', ['userid' => $targetId], ['userid' => $sourceId]);
        
        // Transfer tickets
        update_query('tbltickets', ['userid' => $targetId], ['userid' => $sourceId]);
        
        // Log merge
        insert_query('mod_client_merge_log', [
            'source_id' => $sourceId,
            'target_id' => $targetId,
            'merged_services' => $svcCount['count'],
            'merged_invoices' => $invCount['count'],
            'merged_by' => $_SESSION['adminid']
        ]);
        
        if ($keepTarget) {
            update_query('tblclients', ['status' => 'Inactive'], ['id' => $sourceId]);
        }
        
        echo '<div class="alert alert-success">Clients merged successfully!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Merge Clients</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Source (to be merged)</label><select name="source_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname, " - ", email) as name', '', 'firstname');
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>Target (keep)</label><select name="target_id" class="form-control">';
    mysql_data_seek($clients, 0);
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div></div>
          </div>
          <div class="form-group"><label><input type="checkbox" name="keep_target" value="1" checked /> Deactivate source after merge</label></div>
          <button type="submit" name="merge_clients" class="btn btn-primary">Merge</button>
          </div></form>';
}

function findDuplicateClients() {
    return full_query("SELECT email, COUNT(*) as cnt, GROUP_CONCAT(id) as ids FROM tblclients GROUP BY email HAVING cnt > 1");
}
```