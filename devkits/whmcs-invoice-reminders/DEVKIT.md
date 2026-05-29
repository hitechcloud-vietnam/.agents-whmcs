# WHMCS Invoice Reminders - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_reminders_config() { return ['name' => 'Invoice Reminders', 'description' => 'Automated invoice reminders', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'reminder_days' => ['Type' => 'text', 'Default' => '3,7,14', 'Description' => 'Days before due to send reminders (comma-separated)'],
    'auto_remind' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-send reminders']
]];}

function whmcs_invoice_reminders_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_reminder_schedules` (`id` INT(11) NOT NULL AUTO_INCREMENT, `schedule_name` VARCHAR(255) NOT NULL, `days_before` INT(11) NOT NULL, `template_id` INT(11) DEFAULT NULL, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_reminder_log` (`id` INT(11) NOT NULL AUTO_INCREMENT, `invoice_id` INT(11) NOT NULL, `schedule_id` INT(11) NOT NULL, `sent_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_reminders_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_reminders_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Reminders</h2>';
    if ($_POST['add_schedule']) {
        insert_query('mod_invoice_reminder_schedules', ['schedule_name' => $_POST['schedule_name'], 'days_before' => $_POST['days_before']]);
        echo '<div class="alert alert-success">Schedule created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Reminder Schedule</div><div class="panel-body">
          <div class="form-group"><label>Schedule Name</label><input type="text" name="schedule_name" class="form-control" required /></div>
          <div class="form-group"><label>Days Before Due</label><input type="number" name="days_before" class="form-control" required /></div>
          <button type="submit" name="add_schedule" class="btn btn-primary">Add Schedule</button></div></form>';
    
    $schedules = select_query('mod_invoice_reminder_schedules', '*', '', 'days_before');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Days Before</th><th>Active</th></tr></thead><tbody>';
    while ($s = mysql_fetch_array($schedules)) { echo '<tr><td>' . $s['schedule_name'] . '</td><td>' . $s['days_before'] . '</td><td>' . ($s['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function processInvoiceReminders() {
    $schedules = select_query('mod_invoice_reminder_schedules', '*', ['is_active' => 1]);
    while ($schedule = mysql_fetch_array($schedules)) {
        $dueDate = date('Y-m-d', strtotime('+' . $schedule['days_before'] . ' days'));
        $invoices = select_query('tblinvoices', '*', ["duedate" => $dueDate, "status" => "Unpaid"]);
        
        while ($invoice = mysql_fetch_array($invoices)) {
            $alreadySent = mysql_num_rows(select_query('mod_invoice_reminder_log', 'id', ['invoice_id' => $invoice['id'], 'schedule_id' => $schedule['id']]));
            if (!$alreadySent) {
                $client = mysql_fetch_array(select_query('tblclients', '*', ['id' => $invoice['userid']]));
                send_email($client['email'], 'Invoice Reminder', 'Your invoice #' . $invoice['id'] . ' is due soon.');
                insert_query('mod_invoice_reminder_log', ['invoice_id' => $invoice['id'], 'schedule_id' => $schedule['id']]);
            }
        }
    }
}

add_hook('DailyCronJob', 1, function($vars) { processInvoiceReminders(); });
```