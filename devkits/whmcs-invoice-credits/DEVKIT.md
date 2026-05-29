# WHMCS Invoice Credits - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_credits_config() { return ['name' => 'Invoice Credits', 'description' => 'Manage client credits', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_invoice_credits_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_credits` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `amount` DECIMAL(10,2) NOT NULL, `balance` DECIMAL(10,2) NOT NULL, `description` VARCHAR(255), `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_credits_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_credits_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Credits</h2>';
    if ($_POST['add_credit']) {
        insert_query('mod_client_credits', ['user_id' => $_POST['user_id'], 'amount' => $_POST['amount'], 'balance' => $_POST['amount'], 'description' => $_POST['description']]);
        echo '<div class="alert alert-success">Credit added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Credit</div><div class="panel-body">
          <div class="form-group"><label>Client</label><select name="user_id" class="form-control">';
    $clients = select_query('tblclients', 'id, CONCAT(firstname, " ", lastname) as name', '', 'firstname');
    while ($c = mysql_fetch_array($clients)) { echo '<option value="' . $c['id'] . '">' . $c['name'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Amount</label><input type="number" step="0.01" name="amount" class="form-control" required /></div>
          <div class="form-group"><label>Description</label><input type="text" name="description" class="form-control" /></div>
          <button type="submit" name="add_credit" class="btn btn-primary">Add Credit</button></div></form>';
    
    $credits = select_query('mod_client_credits', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Client</th><th>Amount</th><th>Balance</th><th>Description</th><th>Date</th></tr></thead><tbody>';
    while ($c = mysql_fetch_array($credits)) {
        $u = mysql_fetch_array(select_query('tblclients', 'firstname, lastname', ['id' => $c['user_id']]));
        echo '<tr><td>' . $u['firstname'] . ' ' . $u['lastname'] . '</td><td>$' . $c['amount'] . '</td><td>$' . $c['balance'] . '</td><td>' . $c['description'] . '</td><td>' . $c['created_at'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function getCreditBalance($userId) {
    $result = mysql_fetch_array(full_query("SELECT SUM(balance) as total FROM mod_client_credits WHERE user_id=" . (int)$userId));
    return $result['total'] ?: 0;
}

function applyCredit($userId, $amount) {
    $balance = getCreditBalance($userId);
    if ($balance >= $amount) {
        $credits = select_query('mod_client_credits', '*', ['user_id' => $userId, 'balance>0'], 'created_at');
        while ($credit = mysql_fetch_array($credits) && $amount > 0) {
            $used = min($credit['balance'], $amount);
            update_query('mod_client_credits', ['balance' => $credit['balance'] - $used], ['id' => $credit['id']]);
            $amount -= $used;
        }
        return true;
    }
    return false;
}
```