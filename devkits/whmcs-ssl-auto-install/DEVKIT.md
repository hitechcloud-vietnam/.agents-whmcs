# WHMCS SSL Auto-Install - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ssl_auto_install_config() { return ['name' => 'SSL Auto-Install', 'description' => 'Automatic SSL installation', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'provider' => ['Type' => 'dropdown', 'Options' => 'letsencrypt:Let\'s Encrypt,buypass:Buypass,globalsign:GlobalSign', 'Default' => 'letsencrypt', 'Description' => 'SSL provider'],
    'auto_verify' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-verify domain ownership'],
    'install_path' => ['Type' => 'text', 'Default' => '/etc/ssl/private', 'Description' => 'Certificate install path']
]];}

function whmcs_ssl_auto_install_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_ssl_installations` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain_id` INT(11) NOT NULL, `certificate` TEXT, `private_key` TEXT, `ca_bundle` TEXT, `install_path` VARCHAR(500), `install_status` ENUM('pending','installing','installed','failed') DEFAULT 'pending', `error_message` TEXT, `installed_at` DATETIME, `next_renewal` DATE, PRIMARY KEY (`id`), KEY `domain_id` (`domain_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ssl_auto_install_deactivate() { return ['status' => 'success']; }

function whmcs_ssl_auto_install_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>SSL Auto-Install</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as total, SUM(install_status='installed') as installed, SUM(install_status='failed') as failed FROM mod_ssl_installations"));
    echo '<div class="row"><div class="col-md-4"><div class="panel panel-info"><div class="panel-body"><p>Total: ' . $stats['total'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-body"><p>Installed: ' . $stats['installed'] . '</p></div></div></div>
          <div class="col-md-4"><div class="panel panel-danger"><div class="panel-body"><p>Failed: ' . $stats['failed'] . '</p></div></div></div></div>';
    
    if ($_POST['install_ssl']) {
        $domainId = (int)$_POST['domain_id'];
        $result = initiateSSLInstall($domainId);
        if ($result['success']) {
            echo '<div class="alert alert-success">SSL installation initiated!</div>';
        } else {
            echo '<div class="alert alert-danger">Installation failed: ' . $result['error'] . '</div>';
        }
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Install SSL Certificate</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><select name="domain_id" class="form-control">';
    $domains = select_query('tbldomains', 'id, domain', '', 'domain');
    while ($d = mysql_fetch_array($domains)) { echo '<option value="' . $d['id'] . '">' . $d['domain'] . '</option>'; }
    echo '</select></div>
          <button type="submit" name="install_ssl" class="btn btn-primary">Install Certificate</button></div></form>';
    
    $installs = select_query('mod_ssl_installations i', 'i.*, d.domain', '', 'i.id', 'DESC', '50', 'i.id', 'INNER JOIN tbldomains d ON i.domain_id = d.id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Status</th><th>Installed</th><th>Next Renewal</th></tr></thead><tbody>';
    while ($i = mysql_fetch_array($installs)) { 
        $statusClass = $i['install_status'] === 'installed' ? 'success' : ($i['install_status'] === 'failed' ? 'danger' : 'warning');
        echo '<tr><td>' . $i['domain'] . '</td><td><span class="label label-' . $statusClass . '">' . ucfirst($i['install_status']) . '</span></td><td>' . ($i['installed_at'] ?: '-') . '</td><td>' . ($i['next_renewal'] ?: '-') . '</td></tr>'; 
    }
    echo '</tbody></table></div>';
}

function initiateSSLInstall($domainId) {
    $domain = mysql_fetch_array(select_query('tbldomains', 'domain', ['id' => $domainId]));
    if (!$domain) return ['success' => false, 'error' => 'Domain not found'];
    
    insert_query('mod_ssl_installations', ['domain_id' => $domainId, 'install_status' => 'installing', 'next_renewal' => date('Y-m-d', strtotime('+90 days'))]);
    return ['success' => true, 'message' => 'Installation queued'];
}

add_hook('DailyCronJob', 1, function($vars) {
    $due = select_query('mod_ssl_installations', '*', "install_status='installed' AND next_renewal <= CURDATE()");
    while ($cert = mysql_fetch_array($due)) {
        logActivity("SSL renewal due for domain ID: " . $cert['domain_id']);
    }
});
```