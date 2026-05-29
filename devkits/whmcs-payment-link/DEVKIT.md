# WHMCS Payment Link - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_payment_link_config() { return ['name' => 'Payment Link', 'description' => 'Custom payment links', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_payment_link_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_payment_links` (`id` INT(11) NOT NULL AUTO_INCREMENT, `invoice_id` INT(11) NOT NULL, `link_token` VARCHAR(64) NOT NULL UNIQUE, `expires_at` DATETIME DEFAULT NULL, `is_used` TINYINT(1) DEFAULT 0, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_payment_link_deactivate() { return ['status' => 'success']; }

function whmcs_payment_link_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Payment Links</h2>';
    if ($_POST['generate_link']) {
        $invoiceId = (int)$_POST['invoice_id'];
        $token = bin2hex(random_bytes(16));
        insert_query('mod_payment_links', ['invoice_id' => $invoiceId, 'link_token' => $token, 'expires_at' => date('Y-m-d H:i:s', strtotime('+30 days'))]);
        echo '<div class="alert alert-success">Payment link: <input type="text" value="' . $vars['systemurl'] . '/pay/' . $token . '" readonly class="form-control" /></div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Generate Payment Link</div><div class="panel-body">
          <div class="form-group"><label>Invoice ID</label><input type="number" name="invoice_id" class="form-control" required /></div>
          <button type="submit" name="generate_link" class="btn btn-primary">Generate</button></div></form>';
    
    $links = select_query('mod_payment_links', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Invoice</th><th>Token</th><th>Expires</th><th>Used</th></tr></thead><tbody>';
    while ($l = mysql_fetch_array($links)) { echo '<tr><td>#' . $l['invoice_id'] . '</td><td>' . substr($l['link_token'], 0, 16) . '...</td><td>' . $l['expires_at'] . '</td><td>' . ($l['is_used'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function getPaymentLink($token) {
    return mysql_fetch_array(select_query('mod_payment_links', '*', ['link_token' => $token, 'is_used' => 0]));
}

function markPaymentLinkUsed($token) {
    update_query('mod_payment_links', ['is_used' => 1], ['link_token' => $token]);
}
```