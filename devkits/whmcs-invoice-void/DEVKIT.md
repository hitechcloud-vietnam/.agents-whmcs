# WHMCS Invoice Void - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_void_config() { return ['name' => 'Invoice Void', 'description' => 'Void invoices', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_invoice_void_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_voided_invoices` (`id` INT(11) NOT NULL AUTO_INCREMENT, `invoice_id` INT(11) NOT NULL, `reason` VARCHAR(255), `voided_by` INT(11) DEFAULT NULL, `original_total` DECIMAL(10,2), `voided_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_void_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_void_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Void</h2>';
    if ($_POST['void_invoice']) {
        $invoiceId = (int)$_POST['invoice_id'];
        $invoice = mysql_fetch_array(select_query('tblinvoices', '*', ['id' => $invoiceId]));
        update_query('tblinvoices', ['status' => 'Cancelled'], ['id' => $invoiceId]);
        insert_query('mod_voided_invoices', ['invoice_id' => $invoiceId, 'reason' => $_POST['reason'], 'voided_by' => $_SESSION['adminid'], 'original_total' => $invoice['total']]);
        echo '<div class="alert alert-success">Invoice #' . $invoiceId . ' voided!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading" style="background:#dc3545;color:white;">Void Invoice</div><div class="panel-body">
          <div class="form-group"><label>Invoice ID</label><input type="number" name="invoice_id" class="form-control" required /></div>
          <div class="form-group"><label>Reason</label><input type="text" name="reason" class="form-control" /></div>
          <button type="submit" name="void_invoice" class="btn btn-danger">Void Invoice</button></div></form>';
    
    $voided = select_query('mod_voided_invoices', '*', '', 'voided_at', 'DESC', '20');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Invoice</th><th>Original Total</th><th>Reason</th><th>Voided</th></tr></thead><tbody>';
    while ($v = mysql_fetch_array($voided)) { echo '<tr><td>#' . $v['invoice_id'] . '</td><td>$' . $v['original_total'] . '</td><td>' . $v['reason'] . '</td><td>' . $v['voided_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```