# WHMCS Ticket Assignment - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_assignment_config() { return ['name' => 'Ticket Assignment', 'description' => 'Enhanced ticket assignment', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_assign' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-assign tickets'],
    'load_balancing' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Enable load balancing']
]];}

function whmcs_ticket_assignment_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_assignment_rules` (`id` INT(11) NOT NULL AUTO_INCREMENT, `rule_name` VARCHAR(255) NOT NULL, `conditions` TEXT, `assign_to` VARCHAR(255), `priority` INT(11) DEFAULT 0, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_assignment_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_assignment_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket Assignment</h2>';
    if ($_POST['add_rule']) {
        insert_query('mod_assignment_rules', ['rule_name' => $_POST['rule_name'], 'conditions' => json_encode($_POST['conditions']), 'assign_to' => $_POST['assign_to'], 'priority' => $_POST['priority']]);
        echo '<div class="alert alert-success">Rule created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Assignment Rule</div><div class="panel-body">
          <div class="form-group"><label>Rule Name</label><input type="text" name="rule_name" class="form-control" required /></div>
          <div class="form-group"><label>Conditions (JSON)</label><textarea name="conditions" class="form-control" rows="3"></textarea></div>
          <div class="form-group"><label>Assign To (Admin ID or Department)</label><input type="text" name="assign_to" class="form-control" /></div>
          <div class="form-group"><label>Priority</label><input type="number" name="priority" class="form-control" value="0" /></div>
          <button type="submit" name="add_rule" class="btn btn-primary">Create</button></div></form>';
    
    $rules = select_query('mod_assignment_rules', '*', '', 'priority', 'DESC');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Rule</th><th>Conditions</th><th>Assign To</th><th>Priority</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($rules)) { echo '<tr><td>' . $r['rule_name'] . '</td><td>' . $r['conditions'] . '</td><td>' . $r['assign_to'] . '</td><td>' . $r['priority'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

add_hook('TicketOpen', 1, function($vars) {
    $rules = select_query('mod_assignment_rules', '*', ['is_active' => 1], 'priority', 'DESC');
    while ($rule = mysql_fetch_array($rules)) {
        $match = true;
        foreach (json_decode($rule['conditions'], true) as $k => $v) {
            if (!strpos($vars[$k] ?? '', $v)) $match = false;
        }
        if ($match) return ['assigned_to' => $rule['assign_to']];
    }
});
```