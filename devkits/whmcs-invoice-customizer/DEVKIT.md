# WHMCS Invoice Customizer - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_customizer_config() { return ['name' => 'Invoice Customizer', 'description' => 'Customize invoice appearance', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'primary_color' => ['Type' => 'text', 'Size' => '7', 'Default' => '#007bff', 'Description' => 'Primary color'],
    'show_logo' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show company logo'],
    'footer_text' => ['Type' => 'text', 'Size' => '100', 'Description' => 'Footer text']
]];}

function whmcs_invoice_customizer_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_customizer` (`id` INT(11) NOT NULL AUTO_INCREMENT, `setting_key` VARCHAR(100) NOT NULL, `setting_value` TEXT, PRIMARY KEY (`id`), UNIQUE KEY `setting_key` (`setting_key`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_customizer_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_customizer_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Customizer</h2>';
    if ($_POST['save']) {
        foreach ($_POST as $k => $v) {
            if (in_array($k, ['primary_color', 'show_logo', 'footer_text'])) {
                $exists = mysql_num_rows(select_query('mod_invoice_customizer', 'id', ['setting_key' => $k]));
                if ($exists) update_query('mod_invoice_customizer', ['setting_value' => $v], ['setting_key' => $k]);
                else insert_query('mod_invoice_customizer', ['setting_key' => $k, 'setting_value' => $v]);
            }
        }
        echo '<div class="alert alert-success">Settings saved!</div>';
    }
    $settings = []; $res = select_query('mod_invoice_customizer', '*', ''); while ($s = mysql_fetch_array($res)) $settings[$s['setting_key']] = $s['setting_value'];
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Invoice Settings</div><div class="panel-body">
          <div class="form-group"><label>Primary Color</label><input type="color" name="primary_color" class="form-control" value="' . ($settings['primary_color'] ?: '#007bff') . '" /></div>
          <div class="form-group"><label><input type="checkbox" name="show_logo" value="1" ' . ($settings['show_logo'] ?? '1' == '1' ? 'checked' : '') . ' /> Show Logo</label></div>
          <div class="form-group"><label>Footer Text</label><input type="text" name="footer_text" class="form-control" value="' . htmlspecialchars($settings['footer_text'] ?? '') . '" /></div>
          <button type="submit" name="save" class="btn btn-primary">Save</button></div></form>';
}

add_hook('InvoiceCreation', 1, function($vars) {
    $res = select_query('mod_invoice_customizer', '*', ''); while ($s = mysql_fetch_array($res)) $settings[$s['setting_key']] = $s['setting_value'];
    return ['customizer_settings' => $settings];
});
```