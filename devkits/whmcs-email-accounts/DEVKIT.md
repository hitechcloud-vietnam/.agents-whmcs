# WHMCS Email Accounts - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_email_accounts_config() { return ['name' => 'Email Accounts', 'description' => 'Email account management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'default_quota' => ['Type' => 'text', 'Default' => '1024', 'Description' => 'Default quota (MB)'],
    'max_accounts' => ['Type' => 'text', 'Default' => '5', 'Description' => 'Max accounts per user'],
    'auto_create' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Auto-create with domain']
]];}

function whmcs_email_accounts_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_email_accounts` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `user_id` INT(11) DEFAULT NULL, `email` VARCHAR(255) NOT NULL, `password_hash` VARCHAR(255), `quota` INT(11) DEFAULT 1024, `used_quota` BIGINT DEFAULT 0, `is_active` TINYINT(1) DEFAULT 1, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `email` (`email`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_email_accounts_deactivate() { return ['status' => 'success']; }

function whmcs_email_accounts_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Email Accounts</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(is_active=1) as active, SUM(quota) as total_quota FROM mod_email_accounts"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Total Accounts: ' . $stats['total'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Active: ' . $stats['active'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-warning"><div class="panel-body"><p>Total Quota: ' . number_format($stats['total_quota']) . ' MB</p></div></div></div></div>';
    
    if ($_POST['create_account']) {
        $domainId = (int)$_POST['domain_id'];
        $email = db_escape_string($_POST['email']);
        $quota = (int)$_POST['quota'];
        $password = password_hash($_POST['password'], PASSWORD_DEFAULT);
        insert_query('mod_email_accounts', ['domain_id' => $domainId, 'email' => $email, 'password_hash' => $password, 'quota' => $quota]);
        echo '<div class="alert alert-success">Email account created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Email Account</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Email Address</label><input type="text" name="email" class="form-control" placeholder="username"></div>
          <div class="form-group"><label>Password</label><input type="password" name="password" class="form-control"></div>
          <div class="form-group"><label>Quota (MB)</label><input type="number" name="quota" value="1024" class="form-control"></div>
          <button type="submit" name="create_account" class="btn btn-primary">Create</button></div></form>';
    
    $accounts = select_query('mod_email_accounts a', 'a.*, d.domain', '', 'a.id', 'DESC', '50', 'a.id', 'INNER JOIN tbldomains d ON a.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Email</th><th>Domain</th><th>Quota</th><th>Used</th><th>Status</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($accounts)) { 
        $usage = round(($a['used_quota'] / ($a['quota'] * 1024 * 1024)) * 100, 1);
        echo '<tr><td>' . $a['email'] . '</td><td>' . $a['domain'] . '</td><td>' . $a['quota'] . ' MB</td><td>' . $usage . '%</td><td>' . ($a['is_active'] ? 'Active' : 'Inactive') . '</td></tr>'; 
    }
    echo '</tbody></table></div>';
}
```