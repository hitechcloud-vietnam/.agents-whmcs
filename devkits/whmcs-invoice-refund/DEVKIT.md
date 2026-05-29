# WHMCS Invoice Refund - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_refund_config() { return ['name' => 'Invoice Refund', 'description' => 'Process refunds', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'allow_partial' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow partial refunds']
]];}

function whmcs_invoice_refund_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_refunds` (`id` INT(11) NOT NULL AUTO_INCREMENT, `invoice_id` INT(11) NOT NULL, `amount` DECIMAL(10,2) NOT NULL, `method` VARCHAR(50), `reason` TEXT, `refunded_by` INT(11) DEFAULT NULL, `refunded_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_refund_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_refund_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Refund</h2>';
    if ($_POST['process_refund']) {
        $invoiceId = (int)$_POST['invoice_id'];
        $invoice = mysql_fetch_array(select_query('tblinvoices', '*', ['id' => $invoiceId]));
        $amount = (float)$_POST['amount'];
        
        $refunded = mysql_fetch_array(full_query("SELECT SUM(amount) as total FROM mod_invoice_refunds WHERE invoice_id=" . $invoiceId));
        $remaining = $invoice['total'] - ($refunded['total'] ?: 0);
        
        if ($amount > $remaining) $amount = $remaining;
        
        insert_query('mod_invoice_refunds', ['invoice_id' => $invoiceId, 'amount' => $amount, 'method' => $_POST['method'], 'reason' => $_POST['reason'], 'refunded_by' => $_SESSION['adminid']]);
        echo '<div class="alert alert-success">Refund of $' . $amount . ' processed!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Process Refund</div><div class="panel-body">
          <div class="form-group"><label>Invoice ID</label><input type="number" name="invoice_id" class="form-control" required /></div>
          <div class="form-group"><label>Amount</label><input type="number" step="0.01" name="amount" class="form-control" required /></div>
          <div class="form-group"><label>Method</label><select name="method" class="form-control"><option value="original">Original Payment</option><option value="credit">Account Credit</option><option value="bank">Bank Transfer</option></select></div>
          <div class="form-group"><label>Reason</label><textarea name="reason" class="form-control"></textarea></div>
          <button type="submit" name="process_refund" class="btn btn-primary">Process Refund</button></div></form>';
    
    $refunds = select_query('mod_invoice_refunds', '*', '', 'refunded_at', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Invoice</th><th>Amount</th><th>Method</th><th>Date</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($refunds)) { echo '<tr><td>#' . $r['invoice_id'] . '</td><td>$' . $r['amount'] . '</td><td>' . $r['method'] . '</td><td>' . $r['refunded_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```