# WHMCS Module Localization Workflow

## Description
Add multi-language support to WHMCS modules.

## Steps

### Step 1: Create Language Files
```php
<?php
// languages/english.php

$_LANG = [
    'clicodes_module_name' => 'CLICodes Module',
    'clicodes_settings' => 'Settings',
    'clicodes_save' => 'Save Settings',
    'clicodes_saved' => 'Settings saved successfully.',
    'clicodes_error' => 'An error occurred.',
    
    // Admin area
    'clicodes_dashboard' => 'Dashboard',
    'clicodes_reports' => 'Reports',
    'clicodes_help' => 'Help',
    
    // Client area
    'clicodes_welcome' => 'Welcome to %s',
    'clicodes_manage' => 'Manage Service',
    
    // Messages
    'clicodes_success' => 'Operation completed successfully.',
    'clicodes_loading' => 'Loading...',
    'clicodes_confirm' => 'Are you sure?',
];

return $_LANG;
```

### Step 2: Create Additional Languages
```php
<?php
// languages/vietnamese.php

$_LANG = [
    'clicodes_module_name' => 'Mô-đun CLICodes',
    'clicodes_settings' => 'Cài đặt',
    'clicodes_save' => 'Lưu cài đặt',
    'clicodes_saved' => 'Cài đặt đã được lưu thành công.',
    
    // ... translate all strings
];

return $_LANG;
```

### Step 3: Load Language in Module
```php
<?php
// In main module file

function clicodes_example_output($vars)
{
    $lang = $vars['modulelanguage'] ?? 'english';
    $file = __DIR__ . "/languages/{$lang}.php";
    
    if (file_exists($file)) {
        $_LANG = require $file;
    } else {
        $_LANG = require __DIR__ . '/languages/english.php';
    }
    
    $smarty = new Smarty;
    $smarty->assign('LANG', $_LANG);
    $smarty->assign('lang', $vars['lang']);
    
    return $smarty->fetch(__DIR__ . '/templates/admin/dashboard.tpl');
}
```

### Step 4: Use Language in Templates
```smarty
{*
 * Admin dashboard template
 *}

<h3>{$LANG.clicodes_module_name}</h3>

<form method="post">
    <label>{$LANG.clicodes_settings}</label>
    <input type="text" name="setting" value="{$variables.setting}">
    
    <button type="submit" class="btn btn-primary">
        {$LANG.clicodes_save}
    </button>
</form>

{if $saved}
    <div class="alert alert-success">
        {$LANG.clicodes_saved}
    </div>
{/if}
```

## Supported Languages
| Code | Language |
|------|----------|
| english | English |
| vietnamese | Vietnamese |
| chinese | Chinese |
| spanish | Spanish |
| french | French |
| german | German |

## Tags
- localization
- i18n
- translation
- language