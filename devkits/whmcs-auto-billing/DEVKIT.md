# WHMCS Auto Billing - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_auto_billing_config() { return ['name' => 'Auto Billing', 'description' => 'Automated billing management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'retry_count' => ['Type' => 'text', 'Default' => '3', 'Description' => 'Payment retry count'],
    'retry_days' => ['Type' => 'text', 'Default' => '2', 'Description' => 'Days between retries']
]];}

function whmcs_auto_billing_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_auto_billing_schedules` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `billing_day` INT(11) DEFAULT 1, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_billing_attempts` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `invoice_id` INT(11) DEFAULT NULL, `amount` DECIMAL(10,2), `status` ENUM('pending','success','failed') DEFAULT 'pending', `attempted_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_auto_billing_deactivate() { return ['status' => 'success']; }

function whmcs_auto_billing_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Auto Billing</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(CASE WHEN status='success' THEN 1 ELSE 0 END) as successful FROM mod_billing_attempts"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Total Attempts: ' . $stats['total'] . '</p><p>Successful: ' . $stats['successful'] . '</p></div></div>';
    
    $schedules = select_query('mod_auto_billing_schedules', '*', ['is_active' => 1], 'id', 'DESC', '50');
    echo '<table class="datatable"><thead><tr><th>Client</th><th>Billing Day</th><th>Active</th></tr></thead><tbody>';
    while ($s = mysql_fetch_array($schedules)) {
        $u = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $s['user_id']]));
        echo '<tr><td>' . $u['email'] . '</td><td>Day ' . $s['billing_day'] . '</td><td>Yes</td></tr>';
    }
    echo '</tbody></table></div>';
}

function processAutoBilling() {
    $schedules = select_query('mod_auto_billing_schedules', '*', ['is_active' => 1, 'billing_day' => (int)date('j')]);
    while ($schedule = mysql_fetch_array($schedules)) {
        $invoices = select_query('tblinvoices', '*', ['userid' => $schedule['user_id'], 'status' => 'Unpaid']);
        while ($invoice = mysql_fetch_array($invoices)) {
            $result = localAPI('capturePayment', ['invoiceid' => $invoice['id']]);
            insert_query('mod_billing_attempts', ['user_id' => $schedule['user_id'], 'invoice_id' => $invoice['id'], 'amount' => $invoice['total'], 'status' => $result['result'] == 'success' ? 'success' : 'failed']);
        }
    }
}

add_hook('DailyCronJob', 1, function($vars) { processAutoBilling(); });
```