# WHMCS Ticket SLA - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_sla_config() { return ['name' => 'Ticket SLA', 'description' => 'SLA tracking', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_sla_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_sla_levels` (`id` INT(11) NOT NULL AUTO_INCREMENT, `level_name` VARCHAR(255) NOT NULL, `response_time` INT(11) NOT NULL, `resolution_time` INT(11) NOT NULL, `is_default` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_sla_tracking` (`id` INT(11) NOT NULL AUTO_INCREMENT, `ticket_id` INT(11) NOT NULL, `sla_level_id` INT(11) NOT NULL, `response_due` DATETIME NOT NULL, `resolution_due` DATETIME NOT NULL, `response_at` DATETIME DEFAULT NULL, `resolved_at` DATETIME DEFAULT NULL, `breached` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_sla_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_sla_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket SLA</h2>';
    if ($_POST['add_level']) {
        insert_query('mod_sla_levels', ['level_name' => $_POST['level_name'], 'response_time' => $_POST['response_time'], 'resolution_time' => $_POST['resolution_time'], 'is_default' => $_POST['is_default'] ?? 0]);
        echo '<div class="alert alert-success">SLA level created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add SLA Level</div><div class="panel-body">
          <div class="form-group"><label>Level Name</label><input type="text" name="level_name" class="form-control" required /></div>
          <div class="form-group"><label>Response Time (minutes)</label><input type="number" name="response_time" class="form-control" required /></div>
          <div class="form-group"><label>Resolution Time (hours)</label><input type="number" name="resolution_time" class="form-control" required /></div>
          <div class="form-group"><label><input type="checkbox" name="is_default" value="1" /> Default</label></div>
          <button type="submit" name="add_level" class="btn btn-primary">Create</button></div></form>';
    
    $breaches = mysql_fetch_array(full_query("SELECT COUNT(*) as count FROM mod_sla_tracking WHERE breached=1"));
    echo '<div class="panel panel-danger"><div class="panel-heading">Breached SLAs: ' . $breaches['count'] . '</div></div>';
    
    $levels = select_query('mod_sla_levels', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Level</th><th>Response</th><th>Resolution</th><th>Default</th></tr></thead><tbody>';
    while ($l = mysql_fetch_array($levels)) { echo '<tr><td>' . $l['level_name'] . '</td><td>' . $l['response_time'] . ' min</td><td>' . $l['resolution_time'] . ' hrs</td><td>' . ($l['is_default'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```