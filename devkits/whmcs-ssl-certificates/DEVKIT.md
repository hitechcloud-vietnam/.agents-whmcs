# WHMCS SSL Certificates - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ssl_certificates_config() { return ['name' => 'SSL Certificates', 'description' => 'SSL certificate tracking', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'expiry_warning' => ['Type' => 'text', 'Default' => '30', 'Description' => 'Days before expiry to warn'],
    'auto_renew' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Enable auto-renewal']
]];}

function whmcs_ssl_certificates_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_ssl_certificates` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11), `domain_id` INT(11), `hostname` VARCHAR(255), `order_id` INT(11), `cert_type` VARCHAR(50), `cert_data` TEXT, `issued_at` DATETIME, `expiry_date` DATE, `auto_renew` TINYINT(1) DEFAULT 0, `status` ENUM('pending','active','expired','cancelled') DEFAULT 'pending', `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`), KEY `expiry_date` (`expiry_date`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ssl_certificates_deactivate() { return ['status' => 'success']; }

function whmcs_ssl_certificates_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>SSL Certificates</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(status='active') as active, SUM(expiry_date <= DATE_ADD(CURDATE(), INTERVAL 30 DAY) AND status='active') as expiring FROM mod_ssl_certificates"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Total: ' . $stats['total'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Active: ' . $stats['active'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-danger"><div class="panel-body"><p>Expiring Soon: ' . $stats['expiring'] . '</p></div></div></div></div>';
    
    if ($_POST['add_cert']) {
        $domainId = (int)$_POST['domain_id'];
        $hostname = db_escape_string($_POST['hostname']);
        $expiry = db_escape_string($_POST['expiry_date']);
        $type = db_escape_string($_POST['cert_type']);
        insert_query('mod_ssl_certificates', ['domain_id' => $domainId, 'hostname' => $hostname, 'expiry_date' => $expiry, 'cert_type' => $type, 'status' => 'active', 'issued_at' => date('Y-m-d H:i:s')]);
        echo '<div class="alert alert-success">Certificate added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Certificate</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Hostname</label><input type="text" name="hostname" class="form-control" placeholder="www.example.com"></div>
          <div class="form-group"><label>Certificate Type</label><select name="cert_type" class="form-control"><option value="DV">DV (Domain Validation)</option><option value="OV">OV (Organization Validation)</option><option value="EV">EV (Extended Validation)</option></select></div>
          <div class="form-group"><label>Expiry Date</label><input type="date" name="expiry_date" class="form-control"></div>
          <button type="submit" name="add_cert" class="btn btn-primary">Add</button></div></form>';
    
    $certs = select_query('mod_ssl_certificates c', 'c.*, d.domain', "c.status='active'", 'c.expiry_date', 'ASC', '50', 'c.id', 'INNER JOIN tbldomains d ON c.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Hostname</th><th>Type</th><th>Expiry</th><th>Days Left</th></tr></thead><tbody>';
    while ($c = mysql_fetch_array($certs)) {
        $daysLeft = max(0, (strtotime($c['expiry_date']) - time()) / 86400);
        $class = $daysLeft < 30 ? 'danger' : ($daysLeft < 60 ? 'warning' : 'success');
        echo '<tr><td>' . $c['domain'] . '</td><td>' . $c['hostname'] . '</td><td>' . $c['cert_type'] . '</td><td>' . $c['expiry_date'] . '</td><td class="text-' . $class . '">' . $daysLeft . '</td></tr>'; 
    }
    echo '</tbody></table></div>';
}

add_hook('DailyCronJob', 1, function($vars) {
    $expiring = select_query('mod_ssl_certificates', '*', "expiry_date <= DATE_ADD(CURDATE(), INTERVAL 30 DAY) AND status='active' AND auto_renew=1");
    while ($cert = mysql_fetch_array($expiring)) {
        logActivity("SSL Certificate expiring: " . $cert['hostname']);
    }
});
```