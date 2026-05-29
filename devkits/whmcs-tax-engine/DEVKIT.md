# WHMCS Tax Engine - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_tax_engine_config() { return ['name' => 'Tax Engine', 'description' => 'Advanced tax calculations', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'default_rate' => ['Type' => 'text', 'Default' => '10', 'Description' => 'Default tax rate %']
]];}

function whmcs_tax_engine_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_tax_rules` (`id` INT(11) NOT NULL AUTO_INCREMENT, `rule_name` VARCHAR(255) NOT NULL, `rate` DECIMAL(5,2) NOT NULL, `country` VARCHAR(100) DEFAULT NULL, `state` VARCHAR(100) DEFAULT NULL, `applies_to` ENUM('all','product','category') DEFAULT 'all', `applies_id` INT(11) DEFAULT NULL, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_tax_exemptions` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `exemption_type` VARCHAR(50), `certificate` VARCHAR(255), `valid_until` DATE DEFAULT NULL, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_tax_engine_deactivate() { return ['status' => 'success']; }

function whmcs_tax_engine_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Tax Engine</h2>';
    if ($_POST['add_rule']) {
        insert_query('mod_tax_rules', ['rule_name' => $_POST['rule_name'], 'rate' => $_POST['rate'], 'country' => $_POST['country'] ?: null, 'state' => $_POST['state'] ?: null]);
        echo '<div class="alert alert-success">Tax rule added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Tax Rule</div><div class="panel-body">
          <div class="form-group"><label>Rule Name</label><input type="text" name="rule_name" class="form-control" required /></div>
          <div class="form-group"><label>Rate %</label><input type="number" step="0.01" name="rate" class="form-control" required /></div>
          <div class="form-group"><label>Country</label><input type="text" name="country" class="form-control" /></div>
          <div class="form-group"><label>State</label><input type="text" name="state" class="form-control" /></div>
          <button type="submit" name="add_rule" class="btn btn-primary">Add Rule</button></div></form>';
    
    $rules = select_query('mod_tax_rules', '*', '', 'rate', 'DESC');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Rate</th><th>Country</th><th>State</th></tr></thead><tbody>';
    while ($r = mysql_fetch_array($rules)) { echo '<tr><td>' . $r['rule_name'] . '</td><td>' . $r['rate'] . '%</td><td>' . ($r['country'] ?: 'All') . '</td><td>' . ($r['state'] ?: 'All') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function calculateTax($amount, $userId = null, $country = null, $state = null) {
    $rule = mysql_fetch_array(select_query('mod_tax_rules', '*', ['country' => $country, 'is_active' => 1], 'rate', 'DESC', '1'));
    if (!$rule) {
        $defaultRate = get_config('default_rate') ?: 10;
        return ['tax_rate' => $defaultRate, 'tax_amount' => $amount * ($defaultRate / 100), 'rule' => 'default'];
    }
    return ['tax_rate' => $rule['rate'], 'tax_amount' => $amount * ($rule['rate'] / 100), 'rule' => $rule['rule_name']];
}

function isTaxExempt($userId) {
    $exemption = mysql_fetch_array(select_query('mod_tax_exemptions', '*', ['user_id' => $userId, 'valid_until>' => date('Y-m-d')]));
    return $exemption ? true : false;
}
```