# WHMCS Billing History - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_billing_history_config() { return ['name' => 'Billing History', 'description' => 'Complete billing history', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_billing_history_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_billing_history` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `invoice_id` INT(11), `amount` DECIMAL(10,2) NOT NULL, `payment_method` VARCHAR(50), `status` ENUM('paid','pending','failed','refunded') DEFAULT 'paid', `transaction_id` VARCHAR(100), `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_billing_history_deactivate() { return ['status' => 'success']; }

function whmcs_billing_history_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Billing History</h2>';
    
    if ($_POST['export']) {
        header('Content-Type: text/csv');
        header('Content-Disposition: attachment; filename="billing_history.csv"');
        $output = fopen('php://output', 'w');
        fputcsv($output, ['Date', 'Client', 'Invoice', 'Amount', 'Method', 'Status']);
        $history = select_query('mod_billing_history', '*', '', 'created_at', 'DESC');
        while ($h = mysql_fetch_array($history)) {
            $u = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $h['user_id']]));
            fputcsv($output, [$h['created_at'], $u['email'], $h['invoice_id'], $h['amount'], $h['payment_method'], $h['status']]);
        }
        fclose($output); exit;
    }
    
    echo '<form method="post" style="margin-bottom:20px;"><button type="submit" name="export" class="btn btn-primary">Export CSV</button></form>';
    
    $history = select_query('mod_billing_history', '*', '', 'created_at', 'DESC', '100');
    echo '<table class="datatable"><thead><tr><th>Date</th><th>Client</th><th>Invoice</th><th>Amount</th><th>Method</th><th>Status</th></tr></thead><tbody>';
    while ($h = mysql_fetch_array($history)) {
        $u = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $h['user_id']]));
        echo '<tr><td>' . $h['created_at'] . '</td><td>' . $u['email'] . '</td><td>#' . $h['invoice_id'] . '</td><td>$' . $h['amount'] . '</td><td>' . $h['payment_method'] . '</td><td>' . ucfirst($h['status']) . '</td></tr>';
    }
    echo '</tbody></table></div>';
}
```