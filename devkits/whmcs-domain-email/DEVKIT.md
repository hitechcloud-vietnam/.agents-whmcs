# WHMCS Domain Email - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_email_config() { return ['name' => 'Domain Email', 'description' => 'Domain email service management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'default_quota' => ['Type' => 'text', 'Default' => '1024', 'Description' => 'Default mailbox quota (MB)'],
    'max_accounts' => ['Type' => 'text', 'Default' => '10', 'Description' => 'Max accounts per domain']
]];}

function whmcs_domain_email_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_email_accounts` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `email` VARCHAR(255) NOT NULL, `password` VARCHAR(255), `quota` INT(11) DEFAULT 1024, `is_active` TINYINT(1) DEFAULT 1, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `email` (`email`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_email_aliases` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `source` VARCHAR(255) NOT NULL, `destination` VARCHAR(255) NOT NULL, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_email_deactivate() { return ['status' => 'success']; }

function whmcs_domain_email_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Email</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as accounts FROM mod_domain_email_accounts"));
    $aliases = mysql_fetch_array(full_query("SELECT COUNT(*) as aliases FROM mod_domain_email_aliases"));
    echo '<div class="row"><div class="col-md-6"><div class="panel panel-info"><div class="panel-body"><p>Email Accounts: ' . $stats['accounts'] . '</p></div></div></div>
          <div class="col-md-6"><div class="panel panel-success"><div class="panel-body"><p>Aliases: ' . $aliases['aliases'] . '</p></div></div></div></div>';
    
    if ($_POST['create_account']) {
        $domainId = (int)$_POST['domain_id'];
        $email = db_escape_string($_POST['email']);
        $quota = (int)$_POST['quota'];
        insert_query('mod_domain_email_accounts', ['domain_id' => $domainId, 'email' => $email, 'quota' => $quota]);
        echo '<div class="alert alert-success">Email account created!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Email Account</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Email Address</label><input type="text" name="email" class="form-control" placeholder="user@domain.com"></div>
          <div class="form-group"><label>Quota (MB)</label><input type="number" name="quota" value="1024" class="form-control"></div>
          <button type="submit" name="create_account" class="btn btn-primary">Create</button></div></form>';
    
    $accounts = select_query('mod_domain_email_accounts a', 'a.*, d.domain', '', 'a.id', 'DESC', '50', 'a.id', 'INNER JOIN tbldomains d ON a.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Email</th><th>Domain</th><th>Quota</th><th>Status</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($accounts)) { echo '<tr><td>' . $a['email'] . '</td><td>' . $a['domain'] . '</td><td>' . $a['quota'] . ' MB</td><td>' . ($a['is_active'] ? 'Active' : 'Inactive') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```