# WHMCS Currency Converter - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_currency_converter_config() { return ['name' => 'Currency Converter', 'description' => 'Multi-currency support', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_update' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-update exchange rates']
]];}

function whmcs_currency_converter_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_exchange_rates` (`id` INT(11) NOT NULL AUTO_INCREMENT, `from_currency` VARCHAR(3) NOT NULL, `to_currency` VARCHAR(3) NOT NULL, `rate` DECIMAL(15,8) NOT NULL, `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `currency_pair` (`from_currency`, `to_currency`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_currency_converter_deactivate() { return ['status' => 'success']; }

function whmcs_currency_converter_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Currency Converter</h2>';
    if ($_POST['add_rate']) {
        insert_query('mod_exchange_rates', ['from_currency' => $_POST['from_currency'], 'to_currency' => $_POST['to_currency'], 'rate' => $_POST['rate']]);
        echo '<div class="alert alert-success">Exchange rate added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Exchange Rate</div><div class="panel-body">
          <div class="row">
          <div class="col-md-4"><div class="form-group"><label>From Currency</label><input type="text" name="from_currency" class="form-control" placeholder="USD" maxlength="3" /></div></div>
          <div class="col-md-4"><div class="form-group"><label>To Currency</label><input type="text" name="to_currency" class="form-control" placeholder="EUR" maxlength="3" /></div></div>
          <div class="col-md-4"><div class="form-group"><label>Rate</label><input type="number" step="0.00000001" name="rate" class="form-control" required /></div></div>
          </div>
          <button type="submit" name="add_rate" class="btn btn-primary">Add Rate</button></div></form>';
    
    $rates = select_query('mod_exchange_rates', '*', '', 'from_currency');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>From</th><th>To</th><th>Rate</th><th>Updated</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($rates)) { echo '<tr><td>' . $r['from_currency'] . '</td><td>' . $r['to_currency'] . '</td><td>' . $r['rate'] . '</td><td>' . $r['updated_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function convertCurrency($amount, $from, $to) {
    if ($from == $to) return $amount;
    $rate = mysql_fetch_array(select_query('mod_exchange_rates', 'rate', ['from_currency' => $from, 'to_currency' => $to]));
    return $rate ? $amount * $rate['rate'] : $amount;
}

function formatCurrency($amount, $currency = 'USD') {
    $symbols = ['USD' => '$', 'EUR' => '€', 'GBP' => '£'];
    return ($symbols[$currency] ?? $currency . ' ') . number_format($amount, 2);
}

add_hook('InvoiceCreation', 1, function($vars) {
    $client = mysql_fetch_array(select_query('tblclients', 'currency', ['id' => $vars['userid']]));
    return ['currency' => $client['currency'] ?: 1];
});
```