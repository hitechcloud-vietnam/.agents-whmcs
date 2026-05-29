# WHMCS Domain Renewal - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_renewal_config() { return ['name' => 'Domain Renewal', 'description' => 'Domain renewal management', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_renew' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Enable auto-renewal'],
    'reminder_days' => ['Type' => 'text', 'Default' => '30,14,7', 'Description' => 'Reminder days (comma-separated)']
]];}

function whmcs_domain_renewal_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_renewal_settings` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `domain_id` INT(11) NOT NULL, `auto_renew` TINYINT(1) DEFAULT 0, `renew_years` INT(11) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_renewal_deactivate() { return ['status' => 'success']; }

function whmcs_domain_renewal_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain Renewal</h2>';
    $expiring = full_query("SELECT COUNT(*) as count FROM tbldomains WHERE expirydate BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)");
    $count = mysql_fetch_array($expiring);
    echo '<div class="panel panel-warning"><div class="panel-body"><p>Expiring in 30 days: ' . $count['count'] . '</p></div></div>';
    
    $domains = select_query('tbldomains', '*', "expirydate BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)", 'expirydate', 'ASC', '50');
    echo '<table class="datatable"><thead><tr><th>Domain</th><th>Expiry Date</th><th>Registrant</th></tr></thead><tbody>';
    while ($d = mysql_fetch_array($domains)) { echo '<tr><td>' . $d['domain'] . '</td><td>' . $d['expirydate'] . '</td><td>' . $d['registrant'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function processRenewals() {
    $autoRenew = select_query('mod_domain_renewal_settings', '*', ['auto_renew' => 1]);
    while ($setting = mysql_fetch_array($autoRenew)) {
        $domain = mysql_fetch_array(select_query('tbldomains', '*', ['id' => $setting['domain_id']]));
        if ($domain && strtotime($domain['expirydate']) < strtotime('+30 days')) {
            localAPI('RenewDomain', ['domainid' => $domain['id'], 'years' => $setting['renew_years']]);
        }
    }
}

add_hook('DailyCronJob', 1, function($vars) { processRenewals(); });
```