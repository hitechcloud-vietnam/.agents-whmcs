# WHMCS Client Portal Customizer - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_portal_customizer_config() {
    return [
        'name' => 'Client Portal Customizer',
        'description' => 'Customize the client portal with branding and layout options',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'portal_theme' => ['Type' => 'dropdown', 'Default' => 'default', 
                'Options' => ['default' => 'Default', 'minimal' => 'Minimal', 'modern' => 'Modern', 'dark' => 'Dark'],
                'Description' => 'Portal theme'],
            'custom_logo' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable custom logo'],
            'footer_text' => ['Type' => 'text', 'Size' => '100', 'Description' => 'Custom footer text'],
            'primary_color' => ['Type' => 'text', 'Size' => '7', 'Default' => '#007bff', 'Description' => 'Primary color hex']
        ]
    ];
}

function whmcs_client_portal_customizer_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_portal_customizer_settings` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `setting_key` VARCHAR(100) NOT NULL,
        `setting_value` TEXT,
        PRIMARY KEY (`id`),
        UNIQUE KEY `setting_key` (`setting_key`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    $defaults = ['portal_theme' => 'default', 'primary_color' => '#007bff', 'footer_text' => ''];
    foreach ($defaults as $k => $v) {
        insert_query('mod_portal_customizer_settings', ['setting_key' => $k, 'setting_value' => $v]);
    }
    return ['status' => 'success'];
}

function whmcs_client_portal_customizer_deactivate() { return ['status' => 'success']; }

function whmcs_client_portal_customizer_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Portal Customizer</h2>';
    
    if ($_POST['save_settings']) {
        foreach ($_POST as $k => $v) {
            if (in_array($k, ['portal_theme', 'primary_color', 'footer_text', 'custom_logo'])) {
                update_query('mod_portal_customizer_settings', ['setting_value' => $v], ['setting_key' => $k]);
            }
        }
        echo '<div class="alert alert-success">Settings saved!</div>';
    }
    
    $settings = [];
    $result = select_query('mod_portal_customizer_settings', '*', '');
    while ($s = mysql_fetch_array($result)) { $settings[$s['setting_key']] = $s['setting_value']; }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Portal Settings</div>
          <div class="panel-body">
          <div class="form-group"><label>Theme</label>
          <select name="portal_theme" class="form-control">
          <option value="default"' . ($settings['portal_theme']=='default'?' selected':'') . '>Default</option>
          <option value="minimal"' . ($settings['portal_theme']=='minimal'?' selected':'') . '>Minimal</option>
          <option value="modern"' . ($settings['portal_theme']=='modern'?' selected':'') . '>Modern</option>
          <option value="dark"' . ($settings['portal_theme']=='dark'?' selected':'') . '>Dark</option>
          </select></div>
          <div class="form-group"><label>Primary Color</label>
          <input type="color" name="primary_color" class="form-control" value="' . ($settings['primary_color'] ?: '#007bff') . '" /></div>
          <div class="form-group"><label>Footer Text</label>
          <input type="text" name="footer_text" class="form-control" value="' . htmlspecialchars($settings['footer_text'] ?? '') . '" /></div>
          <div class="form-group"><label><input type="checkbox" name="custom_logo" value="1"' . ($settings['custom_logo']??'1'=='1'?' checked':'') . ' /> Enable Custom Logo</label></div>
          <button type="submit" name="save_settings" class="btn btn-primary">Save</button>
          </div></form></div>';
}

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    $result = select_query('mod_portal_customizer_settings', '*', '');
    while ($s = mysql_fetch_array($result)) { $settings[$s['setting_key']] = $s['setting_value']; }
    return ['portal_theme' => $settings['portal_theme'] ?? 'default', 'primary_color' => $settings['primary_color'] ?? '#007bff'];
});
```