# WHMCS Domain Registration - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_registration_config() { return ['name' => 'Domain Registration', 'description' => 'Domain registration management', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_domain_registration_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_tlds` (`id` INT(11) NOT NULL AUTO_INCREMENT, `tld` VARCHAR(20) NOT NULL UNIQUE, `register_price` DECIMAL(10,2) NOT NULL, `renew_price` DECIMAL(10,2) NOT NULL, `transfer_price` DECIMAL(10,2) DEFAULT NULL, `min_years` INT(11) DEFAULT 1, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_registration_deactivate() { return ['status' => 'success']; }

function whmcs_domain_registration_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Registration</h2>';
    if ($_POST['add_tld']) {
        insert_query('mod_domain_tlds', ['tld' => $_POST['tld'], 'register_price' => $_POST['reg_price'], 'renew_price' => $_POST['renew_price'], 'transfer_price' => $_POST['transfer_price']]);
        echo '<div class="alert alert-success">TLD added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add TLD</div><div class="panel-body">
          <div class="row">
          <div class="col-md-3"><div class="form-group"><label>TLD</label><input type="text" name="tld" class="form-control" placeholder=".com" required /></div></div>
          <div class="col-md-3"><div class="form-group"><label>Register Price</label><input type="number" step="0.01" name="reg_price" class="form-control" required /></div></div>
          <div class="col-md-3"><div class="form-group"><label>Renew Price</label><input type="number" step="0.01" name="renew_price" class="form-control" required /></div></div>
          <div class="col-md-3"><div class="form-group"><label>Transfer Price</label><input type="number" step="0.01" name="transfer_price" class="form-control" /></div></div>
          </div>
          <button type="submit" name="add_tld" class="btn btn-primary">Add TLD</button></div></form>';
    
    $tlds = select_query('mod_domain_tlds', '*', '', 'tld');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>TLD</th><th>Register</th><th>Renew</th><th>Transfer</th><th>Active</th></tr></thead><tbody>';
    while ($t = mysql_fetch_array($tlds)) { echo '<tr><td>' . $t['tld'] . '</td><td>$' . $t['register_price'] . '</td><td>$' . $t['renew_price'] . '</td><td>$' . ($t['transfer_price'] ?: '-') . '</td><td>' . ($t['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function registerDomain($domain, $years = 1) {
    $tld = '.' . substr(strrchr($domain, '.'), 1);
    $tldInfo = mysql_fetch_array(select_query('mod_domain_tlds', '*', ['tld' => $tld, 'is_active' => 1]));
    if (!$tldInfo) return ['error' => 'TLD not available'];
    
    $result = localAPI('RegisterDomain', ['domain' => $domain, 'years' => $years]);
    return $result;
}
```