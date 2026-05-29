# WHMCS Billing Schedule - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_billing_schedule_config() { return ['name' => 'Billing Schedule', 'description' => 'Custom billing schedules', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_billing_schedule_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_billing_schedules` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `schedule_type` ENUM('daily','weekly','monthly','quarterly','annually') DEFAULT 'monthly', `billing_date` INT(11) DEFAULT 1, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_billing_schedule_deactivate() { return ['status' => 'success']; }

function whmcs_billing_schedule_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Billing Schedule</h2>';
    if ($_POST['add_schedule']) {
        insert_query('mod_billing_schedules', ['user_id' => $_POST['user_id'], 'schedule_type' => $_POST['schedule_type'], 'billing_date' => $_POST['billing_date']]);
        echo '<div class="alert alert-success">Schedule created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Billing Schedule</div><div class="panel-body">
          <div class="form-group"><label>Client</label><select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, email', '', 'email'); while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['email'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Schedule Type</label><select name="schedule_type" class="form-control"><option value="daily">Daily</option><option value="weekly">Weekly</option><option value="monthly">Monthly</option><option value="quarterly">Quarterly</option><option value="annually">Annually</option></select></div>
          <div class="form-group"><label>Billing Date</label><input type="number" name="billing_date" class="form-control" value="1" min="1" max="31" /></div>
          <button type="submit" name="add_schedule" class="btn btn-primary">Create Schedule</button></div></form>';
    
    $schedules = select_query('mod_billing_schedules', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Client</th><th>Type</th><th>Date</th><th>Active</th></tr></thead><tbody>';
    while ($s = mysql_fetch_array($schedules)) {
        $u = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $s['user_id']]));
        echo '<tr><td>' . $u['email'] . '</td><td>' . ucfirst($s['schedule_type']) . '</td><td>' . $s['billing_date'] . '</td><td>' . ($s['is_active'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}
```