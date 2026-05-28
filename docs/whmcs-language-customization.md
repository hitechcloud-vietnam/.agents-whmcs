# WHMCS Language and Phrase Customization

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-multilanguage-support`, `whmcs-internationalization`, `module-localization-guide`

---

## Overview

WHMCS uses a phrase-based localization system that allows you to customize all text strings throughout the application. This guide covers phrase files, custom translations, module localization, and best practices for language customization.

---

## Language Files Structure

### Directory Structure

```
lang/
├── admin/
│   ├── english.php
│   ├── spanish.php
│   └── ... (admin language files)
└── client/
    ├── english.php
    ├── spanish.php
    └── ... (client area language files)
```

### File Format

```php
<?php
/**
 * WHMCS Language Phrases
 * Language: English
 */

$_LANG = [
    // General
    'welcome' => 'Welcome',
    'logout' => 'Logout',
    'login' => 'Login',

    // Navigation
    'navhome' => 'Home',
    'navservices' => 'Services',
    'navdomains' => 'Domains',
    'navbilling' => 'Billing',
    'navsupport' => 'Support',

    // Services
    'ordernew' => 'Order New Services',
    'myservices' => 'My Services',
    'existingorders' => 'Existing Orders',

    // Error messages
    'error' => 'Error',
    'success' => 'Success',
    'invalidemail' => 'Please enter a valid email address',

    // ...
];
```

---

## Custom Language Overrides

### Creating Custom Language File

```php
<?php
/**
 * Custom English Overrides
 * Place this in: lang/custom/english.php
 */

// Override existing phrases
$_LANG['welcome'] = 'Welcome to Our Portal';
$_LANG['myservices'] = 'My Products & Services';

// Add new phrases (use unique keys)
$_LANG['custom']['premium_support'] = 'Premium Support Available';
$_LANG['custom']['sla_badge'] = 'SLA Guaranteed';
```

### Module Language Files

```
modules/
└── addons/
    └── my_module/
        ├── my_module.php
        └── lang/
            ├── english.php
            ├── spanish.php
            └── ... (other languages)
```

```php
<?php
// modules/addons/my_module/lang/english.php

$_MODULE = [
    'my_module' => 'My Module',
    'my_module_description' => 'A powerful module',
    'settings' => 'Module Settings',
    'save_settings' => 'Save Settings',
    'settings_saved' => 'Settings saved successfully',

    // Configuration labels
    'config_api_key' => 'API Key',
    'config_webhook_url' => 'Webhook URL',
    'config_debug_mode' => 'Debug Mode',

    // Messages
    'error_api_connection' => 'Failed to connect to API',
    'success_data_synced' => 'Data synchronized successfully',
];
```

---

## Phrase Functions

### Basic Usage

```php
<?php
// In PHP files
echo $_LANG['welcome'];

// In templates (Smarty)
{$LANG.welcome}

// With sprintf placeholders
$_LANG['greeting'] = 'Hello %s, welcome back!';
sprintf($_LANG['greeting'], $firstName);
```

### Translation Functions

```php
<?php
/**
 * Lang::get() - Get translated phrase
 */
use WHMCS\Language\Lang;

$phrase = Lang::trans('welcome');
$phrase = Lang::trans('greeting', [
    '{name}' => $userName,
    '{date}' => date('Y-m-d'),
]);

// In templates
{Lang::get('welcome')}
{!! Lang::trans('html_content') !!}

/**
 * Admin lang() helper
 */
lang('save');

// With variables
lang('invoice_total', [':amount' => $total]);

/**
 * Client area trans() helper
 */
trans('myservices');
```

### Pluralization

```php
<?php
// lang/english.php
$_LANG['items_count'] = '%d item(s)';
$_LANG['services_active'] = [
    'zero' => 'No active services',
    'one' => '%d active service',
    'other' => '%d active services',
];

// Usage
$count = count($services);
$message = sprintf(
    $_LANG['services_active'],
    $count
);

// Or using pluralize helper
$message = pluralize($count, 'service', 'services');
```

---

## Dynamic Translations

### Runtime Phrase Modification

```php
<?php
/**
 * Modify phrases at runtime
 */

// Hook: Load custom phrases
add_hook('AfterLanguageLoad', 1, function($vars) {
    // Check if user has premium status
    if ($_SESSION['uid']) {
        $client = Capsule::table('tblclients')
            ->where('id', $_SESSION['uid'])
            ->first();

        if ($client->premium) {
            // Add premium-specific phrases
            $_LANG['support_priority'] = 'Priority Support';
            $_LANG['support_sla'] = '4-Hour Response SLA';
        }
    }
});
```

### Conditional Translations

```php
<?php
// hooks.php

// Change greeting based on time of day
add_hook('ClientAreaPage', 1, function($vars) {
    $hour = (int) date('H');

    if ($hour < 12) {
        $_LANG['daily_greeting'] = 'Good Morning';
    } elseif ($hour < 18) {
        $_LANG['daily_greeting'] = 'Good Afternoon';
    } else {
        $_LANG['daily_greeting'] = 'Good Evening';
    }
});

// Brand-specific text
add_hook('AfterLanguageLoad', 1, function($vars) {
    $brand = Capsule::table('tblconfiguration')
        ->where('setting', 'CompanyName')
        ->value('value');

    $_LANG['support_tagline'] = "$brand - Trusted by thousands";
});
```

---

## Module Translation Integration

### Module Translation Class

```php
<?php
// modules/addons/my_module/MyModule.php

namespace WHMCS\Module\Addon\MyModule;

class MyModule
{
    private string $moduleName = 'my_module';

    /**
     * Get translated phrase
     */
    private function translate(string $key, array $vars = []): string
    {
        $phrase = Lang::trans(
            $this->moduleName . '.' . $key,
            $vars
        );

        // Fallback if not found
        if ($phrase === $this->moduleName . '.' . $key) {
            $phrase = $this->getDefaultPhrase($key);
        }

        return $phrase;
    }

    /**
     * Get default phrase (English fallback)
     */
    private function getDefaultPhrase(string $key): string
    {
        $defaults = [
            'title' => 'My Module',
            'description' => 'Module description',
            'save_success' => 'Settings saved successfully',
            'save_error' => 'Failed to save settings',
        ];

        return $defaults[$key] ?? $key;
    }

    /**
     * Output with translation
     */
    public function renderSettings(): void
    {
        $vars = [
            'moduleTitle' => $this->translate('title'),
            'saveButton' => $this->translate('save'),
        ];

        echo $this->getTemplate()->render('settings', $vars);
    }
}
```

### Translation File Structure

```
modules/addons/my_module/
├── MyModule.php
└── lang/
    ├── english.php
    ├── spanish.php
    ├── german.php
    └── french.php
```

```php
<?php
// modules/addons/my_module/lang/spanish.php

$_MODULE = [
    // Module info
    'my_module.title' => 'Mi Módulo',
    'my_module.description' => 'Un módulo potente',

    // Settings
    'my_module.settings' => 'Configuración',
    'my_module.save' => 'Guardar',
    'my_module.reset' => 'Restablecer',

    // Labels
    'my_module.api_key' => 'Clave API',
    'my_module.webhook_url' => 'URL del Webhook',

    // Messages
    'my_module.save_success' => 'Configuración guardada',
    'my_module.save_error' => 'Error al guardar',
    'my_module.test_success' => 'Conexión exitosa',
    'my_module.test_error' => 'Error de conexión',
];
```

---

## Email Template Variables

### Language Variables in Emails

```php
<?php
// Email templates can use language variables
// In email template content:

{$LANG.welcome}                    // "Welcome to {$company_name}"
{$LANG.dear} {$client.firstname}  // "Dear John"
{$LANG.invoice_total_text} {$invoice.total} // "Invoice Total: $100.00"

// Custom variables
{$custom_message}
```

### Email Language Handling

```php
<?php
/**
 * Send email in client's language
 */
function sendLocalizedEmail(
    string $templateName,
    int $clientId,
    array $mergeFields = []
): bool {

    // Get client's language
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();

    $language = $client->language ?: 'english';

    // Load language file
    Language::loadLang($language, null, ROOTDIR . '/lang/');

    // Get email template
    $template = Capsule::table('tblemailtemplates')
        ->where('name', $templateName)
        ->where('language', $language)
        ->first();

    if (!$template) {
        // Fallback to English
        $template = Capsule::table('tblemailtemplates')
            ->where('name', $templateName)
            ->where('language', '')
            ->first();
    }

    // Replace variables
    $subject = replaceKeywords($template->subject, $mergeFields);
    $body = replaceKeywords($template->message, $mergeFields);

    // Send email
    return send_email($clientId, $subject, $body);
}
```

---

## RTL Language Support

### RTL Configuration

```php
<?php
// config.php or setup

// Mark languages as RTL
$r languages = [
    'arabic' => true,
    'hebrew' => true,
    'persian' => true,
    'urdu' => true,
];

/**
 * Check if current language is RTL
 */
function isRtlLanguage(string $language): bool
{
    $rtlLanguages = ['arabic', 'hebrew', 'persian', 'urdu'];
    return in_array(strtolower($language), $rtlLanguages);
}

/**
 * Get direction attribute
 */
function getDirection(string $language): string
{
    return isRtlLanguage($language) ? 'rtl' : 'ltr';
}
```

### RTL Styles

```css
/* RTL-aware styles */
body[dir="rtl"] .sidebar {
    left: auto;
    right: 0;
}

body[dir="rtl"] .nav-icon {
    margin-right: 0;
    margin-left: 0.5rem;
}

body[dir="rtl"] .text-left {
    text-align: right;
}

body[dir="rtl"] .text-right {
    text-align: left;
}
```

---

## Translation Workflow

### Export/Import Phrases

```php
<?php
/**
 * Export phrases to JSON
 */
function exportPhrases(string $language, string $scope = 'client'): array
{
    $filePath = sprintf(
        ROOTDIR . '/lang/%s/%s.php',
        $scope,
        $language
    );

    if (!file_exists($filePath)) {
        throw new \Exception("Language file not found");
    }

    include $filePath;

    return [
        'language' => $language,
        'scope' => $scope,
        'phrases' => $_LANG,
        'exported_at' => date('Y-m-d H:i:s'),
    ];
}

/**
 * Import phrases from JSON
 */
function importPhrases(string $json): bool
{
    $data = json_decode($json, true);

    if (!$data || !isset($data['phrases'])) {
        throw new \Exception("Invalid format");
    }

    $content = "<?php\n/**\n * Generated by import\n */\n\n";
    $content .= '$_LANG = ' . var_export($data['phrases'], true) . ';';

    $filePath = sprintf(
        ROOTDIR . '/lang/%s/%s.php',
        $data['scope'],
        $data['language']
    );

    return file_put_contents($filePath, $content) !== false;
}
```

### Translation API Integration

```php
<?php
/**
 * Send phrases to translation service
 */
function sendToTranslationService(
    string $sourceLanguage,
    string $targetLanguage,
    array $phrases
): array {

    $api = new TranslationApiClient(getenv('TRANSLATION_API_KEY'));

    $result = $api->translate([
        'source' => $sourceLanguage,
        'target' => $targetLanguage,
        'texts' => array_values($phrases),
    ]);

    return array_combine(
        array_keys($phrases),
        $result['translations']
    );
}
```

---

## Best Practices

1. **Use descriptive keys** - `account_settings_title` not `ast`
2. **Group related phrases** - Use namespacing: `module_name.action`
3. **Document placeholders** - Clearly mark variables like `{:name}` or `%s`
4. **Avoid hardcoded text** - Always use phrase keys
5. **Plan for expansion** - Use arrays for pluralization
6. **Test RTL support** - Verify RTL layouts work correctly
7. **Version control** - Keep language files in git

---

## Common Phrase Patterns

```php
<?php
// Status messages
$_LANG['status']['active'] = 'Active';
$_LANG['status']['suspended'] = 'Suspended';
$_LANG['status']['terminated'] = 'Terminated';
$_LANG['status']['pending'] = 'Pending';
$_LANG['status']['cancelled'] = 'Cancelled';

// Actions
$_LANG['action']['view'] = 'View';
$_LANG['action']['edit'] = 'Edit';
$_LANG['action']['delete'] = 'Delete';
$_LANG['action']['cancel'] = 'Cancel';
$_LANG['action']['confirm'] = 'Confirm';
$_LANG['action']['submit'] = 'Submit';

// Validation messages
$_LANG['validation']['required'] = 'This field is required';
$_LANG['validation']['email_invalid'] = 'Please enter a valid email';
$_LANG['validation']['password_length'] = 'Password must be at least 8 characters';
```

---

## Related Documentation

- [Internationalization Guide](internationalization-guide.md)
- [Module Localization Guide](module-localization-guide.md)
- [Email Template Variables](email-template-variables.md)
- [Multi-Language Support](../skills/whmcs-multilanguage-support.md)
