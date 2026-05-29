# WHMCS Invoice Collection - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_collection_config() { return ['name' => 'Invoice Collection', 'description' => 'Invoice collection workflows', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_invoice_collection_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_collection_cases` (`id` INT(11) NOT NULL AUTO_INCREMENT, `invoice_id` INT(11) NOT NULL, `stage` ENUM('initial','reminder','warning','final','closed') DEFAULT 'initial', `assigned_to` INT(11) DEFAULT NULL, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_collection_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_collection_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Collection</h2>';
    $overdue = select_query('tblinvoices', 'COUNT(*) as count', ["status" => "Unpaid", "duedate<" => date('Y-m-d')]);
    $count = mysql_fetch_array($overdue);
    echo '<div class="panel panel-info"><div class="panel-body"><p>Overdue Invoices: ' . $count['count'] . '</p></div></div>';
    
    $cases = select_query('mod_collection_cases mc', 'mc.*, i.total as amount', '', 'mc.created_at', 'DESC', '20', 'mc.id', 'INNER JOIN tblinvoices i ON mc.invoice_id = i.id');
    echo '<table class="datatable"><thead><tr><th>Invoice</th><th>Amount</th><th>Stage</th><th>Created</th></tr></thead><tbody>';
    while ($c = mysql_fetch_array($cases)) { echo '<tr><td>#' . $c['invoice_id'] . '</td><td>$' . $c['amount'] . '</td><td>' . ucfirst($c['stage']) . '</td><td>' . $c['created_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

add_hook('DailyCronJob', 1, function($vars) {
    $overdueInvoices = select_query('tblinvoices', '*', ["status" => "Unpaid", "duedate<" => date('Y-m-d')]);
    while ($inv = mysql_fetch_array($overdueInvoices)) {
        $existing = mysql_num_rows(select_query('mod_collection_cases', 'id', ['invoice_id' => $inv['id']]));
        if (!$existing) insert_query('mod_collection_cases', ['invoice_id' => $inv['id'], 'stage' => 'initial']);
    }
});
```