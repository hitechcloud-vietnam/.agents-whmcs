# WHMCS Client Filter - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_filter_config() {
    return ['name' => 'Client Filter', 'description' => 'Advanced client filtering', 'author' => 'DevKit Generator', 'version' => '1.0.0'];
}

function whmcs_client_filter_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_filter_presets` (
        `id` INT(11) NOT NULL AUTO_INCREMENT, `preset_name` VARCHAR(255) NOT NULL, `filters` TEXT, `staff_id` INT(11) DEFAULT NULL, PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_filter_deactivate() { return ['status' => 'success']; }

function whmcs_client_filter_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Filter Presets</h2>';
    
    if ($_POST['save_preset']) {
        insert_query('mod_client_filter_presets', [
            'preset_name' => $_POST['preset_name'],
            'filters' => json_encode($_POST['filters'] ?? []),
            'staff_id' => $_SESSION['adminid']
        ]);
        echo '<div class="alert alert-success">Preset saved!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Save Filter Preset</div>
          <div class="panel-body">
          <div class="form-group"><label>Preset Name</label><input type="text" name="preset_name" class="form-control" required /></div>
          <div class="form-group"><label>Filters (JSON)</label><textarea name="filters" class="form-control" rows="3">{"status":"Active","has_invoices":true}</textarea></div>
          <button type="submit" name="save_preset" class="btn btn-primary">Save Preset</button>
          </div></form>';
    
    $presets = select_query('mod_client_filter_presets', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Filters</th></tr></thead><tbody>';
    while ($p = mysql_fetch_array($presets)) {
        echo '<tr><td>' . $p['preset_name'] . '</td><td>' . $p['filters'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function applyClientFilter($filters) {
    $where = ['1=1'];
    if (!empty($filters['status'])) $where[] = "status = '" . db_escape($filters['status']) . "'";
    if (!empty($filters['has_invoices'])) $where[] = "id IN (SELECT userid FROM tblinvoices WHERE userid = tblclients.id)";
    return select_query('tblclients', '*', implode(' AND ', $where));
}
```