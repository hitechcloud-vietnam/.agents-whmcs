# WHMCS Invoice Split - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_split_config() { return ['name' => 'Invoice Split', 'description' => 'Split invoices', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_invoice_split_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_split_invoices` (`id` INT(11) NOT NULL AUTO_INCREMENT, `original_invoice_id` INT(11) NOT NULL, `new_invoice_ids` TEXT, `split_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_split_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_split_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Split</h2>';
    if ($_POST['split_invoice']) {
        $invoiceId = (int)$_POST['invoice_id'];
        $splits = json_decode($_POST['splits'], true);
        $invoice = mysql_fetch_array(select_query('tblinvoices', '*', ['id' => $invoiceId]));
        
        $newIds = [];
        foreach ($splits as $split) {
            $newId = createInvoice($invoice['userid'], [['description' => $split['description'], 'qty' => 1, 'price' => $split['amount']]]);
            $newIds[] = $newId;
        }
        
        insert_query('mod_split_invoices', ['original_invoice_id' => $invoiceId, 'new_invoice_ids' => implode(',', $newIds)]);
        echo '<div class="alert alert-success">Invoice split into: ' . implode(', ', $newIds) . '</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Split Invoice</div><div class="panel-body">
          <div class="form-group"><label>Invoice ID</label><input type="number" name="invoice_id" class="form-control" required /></div>
          <div class="form-group"><label>Split Amounts (JSON)</label><textarea name="splits" class="form-control" rows="4" placeholder=\'[{"description":"Part 1","amount":50},{"description":"Part 2","amount":50}]\'></textarea></div>
          <button type="submit" name="split_invoice" class="btn btn-primary">Split Invoice</button></div></form>';
}

function createInvoice($userId, $items) {
    $result = localAPI('CreateInvoice', ['userid' => $userId, 'date' => date('Y-m-d')]);
    return $result['invoiceid'] ?? 0;
}
```