# WHMCS Invoice Generator - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_generator_config() { return ['name' => 'Invoice Generator', 'description' => 'Custom invoice generation', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'default_tax' => ['Type' => 'text', 'Default' => '10', 'Description' => 'Default tax rate %'],
    'auto_send' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-send invoices']
]];}

function whmcs_invoice_generator_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_templates` (`id` INT(11) NOT NULL AUTO_INCREMENT, `template_name` VARCHAR(255) NOT NULL, `template_data` TEXT, `is_default` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_generator_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_generator_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Generator</h2>';
    if ($_POST['generate_invoice']) {
        $userId = (int)$_POST['user_id'];
        $items = json_decode($_POST['items'], true);
        $subtotal = 0;
        foreach ($items as $item) { $subtotal += ($item['qty'] * $item['price']); }
        $tax = $subtotal * ((float)($_POST['tax_rate'] ?: 10) / 100);
        $total = $subtotal + $tax;
        
        $invoiceId = createInvoice($userId, $items, $subtotal, $tax, $total);
        echo '<div class="alert alert-success">Invoice #' . $invoiceId . ' created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Generate Invoice</div><div class="panel-body">
          <div class="form-group"><label>Client</label><select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname) as name', '', 'firstname');
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Items (JSON)</label><textarea name="items" class="form-control" rows="5" placeholder=\'[{"description":"Item 1","qty":1,"price":100}]\'></textarea></div>
          <div class="form-group"><label>Tax Rate %</label><input type="number" step="0.01" name="tax_rate" class="form-control" value="10" /></div>
          <button type="submit" name="generate_invoice" class="btn btn-primary">Generate Invoice</button></div></form>';
}

function createInvoice($userId, $items, $subtotal, $tax, $total) {
    $result = localAPI('CreateInvoice', ['userid' => $userId, 'date' => date('Y-m-d'), 'duedate' => date('Y-m-d', strtotime('+14 days')), 'itemdescription1' => $items[0]['description'], 'itemamount1' => $items[0]['price'] * $items[0]['qty']]);
    return $result['invoiceid'] ?? 0;
}
```