# WHMCS Domain WHOIS - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_domain_whois_config() { return ['name' => 'Domain WHOIS', 'description' => 'WHOIS lookup and display', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'cache_time' => ['Type' => 'text', 'Default' => '3600', 'Description' => 'Cache WHOIS data (seconds)'],
    'show_privacy' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show privacy info']
]];}

function whmcs_domain_whois_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_domain_whois_cache` (`id` INT(11) NOT NULL AUTO_INCREMENT, `domain` VARCHAR(255) NOT NULL, `registrar` VARCHAR(255), `registrant_name` VARCHAR(255), `registrant_email` VARCHAR(255), `nameservers` TEXT, `expiry_date` DATE, `status` VARCHAR(100), `raw_data` TEXT, `cached_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), UNIQUE KEY `domain` (`domain`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_domain_whois_deactivate() { return ['status' => 'success']; }

function whmcs_domain_whois_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Domain WHOIS</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as cached FROM mod_domain_whois_cache"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Cached WHOIS Records: ' . $stats['cached'] . '</p></div></div>';
    
    if ($_POST['lookup_domain']) {
        $domain = db_escape_string($_POST['domain']);
        $whoisData = lookupWhois($domain);
        if ($whoisData) {
            $existing = mysql_fetch_array(select_query('mod_domain_whois_cache', 'id', ['domain' => $domain]));
            $data = ['domain' => $domain, 'registrar' => $whoisData['registrar'], 'nameservers' => json_encode($whoisData['nameservers']), 'expiry_date' => $whoisData['expiry'], 'status' => json_encode($whoisData['status']), 'cached_at' => date('Y-m-d H:i:s')];
            if ($existing) {
                update_query('mod_domain_whois_cache', $data, ['domain' => $domain]);
            } else {
                insert_query('mod_domain_whois_cache', $data);
            }
            echo '<div class="alert alert-success">WHOIS lookup completed!</div>';
        } else {
            echo '<div class="alert alert-warning">Could not retrieve WHOIS data.</div>';
        }
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">WHOIS Lookup</div><div class="panel-body">
          <div class="form-group"><label>Domain</label><input type="text" name="domain" class="form-control" placeholder="example.com"></div>
          <button type="submit" name="lookup_domain" class="btn btn-primary">Lookup</button></div></form>';
    
    $cache = select_query('mod_domain_whois_cache', '*', '', 'cached_at', 'DESC', '50');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Domain</th><th>Registrar</th><th>Expiry</th><th>Cached</th></tr></thead><tbody>';
    while ($w = mysql_fetch_array($cache)) { echo '<tr><td>' . $w['domain'] . '</td><td>' . $w['registrar'] . '</td><td>' . $w['expiry_date'] . '</td><td>' . $w['cached_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function lookupWhois($domain) {
    $socket = fsockopen('whois.verisign-grs.com', 43, $errno, $errstr, 10);
    if (!$socket) return null;
    fwrite($socket, $domain . "\r\n");
    $response = '';
    while (!feof($socket)) { $response .= fgets($socket, 1024); }
    fclose($socket);
    return parseWhoisResponse($response);
}

function parseWhoisResponse($data) {
    $result = ['registrar' => '', 'nameservers' => [], 'expiry' => null, 'status' => []];
    if (preg_match('/Registrar:\s*(.+)/i', $data, $m)) $result['registrar'] = trim($m[1]);
    if (preg_match('/Expiration Date:\s*(.+)/i', $data, $m)) $result['expiry'] = trim($m[1]);
    preg_match_all('/Name Server:\s*(.+)/i', $data, $ns);
    $result['nameservers'] = array_map('trim', $ns[1]);
    return $result;
}
```