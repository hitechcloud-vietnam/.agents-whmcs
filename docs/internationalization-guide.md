# WHMCS Internationalization Guide

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `email-template-variables`, `smarty-template-reference`

## Overview

WHMCS supports multiple languages and localization. This guide covers language file management, translation workflows, RTL support, and best practices for internationalization.

## Language File Structure

### Default Language Files

```
whmcs/
├── lang/
│   ├── english.php          # Default language
│   ├── english british.php   # UK English variant
│   ├── deutsch.php          # German
│   ├── francais.php         # French
│   ├── espanol.php          # Spanish
│   └── ... (other languages)
```

### Language File Format

```php
<?php
/**
 * WHMCS Language File
 * Language: English
 * Version: 8.x
 */

$_LANG = [
    // Page Titles
    'pagetitlehome' => 'Home',
    'pagetitlelogin' => 'Client Login',
    'pagetitleregister' => 'Register',

    // Navigation
    'navhome' => 'Home',
    'navlogin' => 'Login',
    'navlogout' => 'Logout',
    'navaccount' => 'My Account',
    'navservices' => 'Services',
    'navdomains' => 'Domains',
    'navbilling' => 'Billing',
    'navsupport' => 'Support',

    // Client Area
    'clientarea' => 'Client Area',
    'welcomeback' => 'Welcome back, %s',

    // Common
    'save' => 'Save',
    'cancel' => 'Cancel',
    'delete' => 'Delete',
    'edit' => 'Edit',
    'submit' => 'Submit',
    'loading' => 'Loading...',

    // Errors
    'error' => 'Error',
    'success' => 'Success',
    'warning' => 'Warning',
    'info' => 'Information',

    // Date/Time
    'dateformat' => 'm/d/Y',
    'timeformat' => 'g:i A',
];
```

## Custom Language Variables

### Adding Custom Strings

```php
<?php
// Add to your custom lang file or override in hooks

// Method 1: Hook to extend language
add_hook('AfterLanguageLoad', 1, function($vars) {
    return [
        'custom_feature_title' => 'My Custom Feature',
        'custom_feature_desc' => 'Description of my feature',
        'custom_button_text' => 'Get Started',
    ];
});

// Method 2: Direct override (less recommended)
// Add to lang/custom.php
$_LANG['custom_string'] = 'Custom translation';
```

### Using Variables in Strings

```php
<?php
// Basic variable
$_LANG['welcome_message'] = 'Welcome, %s!';
// Usage: sprintf($_LANG['welcome_message'], $username);

// Multiple variables
$_LANG['order_summary'] = 'Order #%d for %s totaling %s';
// Usage: sprintf($_LANG['order_summary'], $orderId, $clientName, $amount);

// Named placeholders
$_LANG['renewal_notice'] = 'Your {domain} will renew on {date}';
// Usage: str_replace(['{domain}', '{date}'], [$domain, $date], $_LANG['renewal_notice']);
```

## Multi-Language Templates

### Template Language Support

```smarty
{* Check current language *}
{$LANG.direction} {* ltr or rtl *}
{$LANG.locale}    {* en_US, de_DE, etc. *}

{* Conditional content by language *}
{if $language eq 'german'}
    <p>Willkommen bei unserem Service!</p>
{elseif $language eq 'french'}
    <p>Bienvenue dans notre service!</p>
{else}
    <p>Welcome to our service!</p>
{/if}

{* Use language strings *}
<h1>{$LANG.pagetitlehome}</h1>
<button>{$LANG.submit}</button>
```

### RTL (Right-to-Left) Support

```smarty
{* Detect RTL languages *}
{if $LANG.direction eq 'rtl'}
    <html dir="rtl" lang="{$LANG.locale}">
{/if}

{* RTL-aware styling *}
<div class="{if $LANG.direction eq 'rtl'}mr-auto{else}ml-auto{/if}">
    Content aligned appropriately
</div>

{* RTL-aware margins/padding *}
<div style="margin-{if $LANG.direction eq 'rtl'}right{else}left{/if}: 10px;">
    RTL-aware margin
</div>
```

### RTL CSS Classes

```css
/* rtl.css - Right-to-Left stylesheet */

[dir="rtl"] .pull-left {
    float: right !important;
}

[dir="rtl"] .pull-right {
    float: left !important;
}

[dir="rtl"] .text-left {
    text-align: right !important;
}

[dir="rtl"] .text-right {
    text-align: left !important;
}

[dir="rtl"] .mr-auto {
    margin-right: auto !important;
    margin-left: initial !important;
}

[dir="rtl"] .ml-auto {
    margin-left: auto !important;
    margin-right: initial !important;
}

[dir="rtl"] .pr-3 {
    padding-right: 0 !important;
    padding-left: 1rem !important;
}

[dir="rtl"] .pl-3 {
    padding-left: 0 !important;
    padding-right: 1rem !important;
}
```

## Currency Localization

### Multiple Currencies

```php
// includes/hooks/currency_localization.php

add_hook('AfterCurrencyLoad', 1, function($vars) {
    // Currency formatting based on locale
    $currencyFormats = [
        'en_US' => [
            'decimal' => '.',
            'thousands' => ',',
            'symbol' => '$',
            'position' => 'before',
        ],
        'de_DE' => [
            'decimal' => ',',
            'thousands' => '.',
            'symbol' => '€',
            'position' => 'after',
        ],
        'fr_FR' => [
            'decimal' => ',',
            'thousands' => ' ',
            'symbol' => '€',
            'position' => 'after',
        ],
    ];

    return [
        'currency_format' => $currencyFormats[$vars['locale']] ?? $currencyFormats['en_US'],
    ];
});

/**
 * Format currency for display
 */
function formatCurrencyLocal(float $amount, string $currency, string $locale): string
{
    return NumberFormatter::create($locale, NumberFormatter::CURRENCY)
        ->formatCurrency($amount, $currency);
}

// Usage
$formatted = formatCurrencyLocal(1234.56, 'EUR', 'de_DE');
// Output: 1.234,56 €
```

## Date/Time Localization

### Date Formatting

```php
<?php
// includes/functions_localization.php

/**
 * Format date according to locale
 */
function formatDateLocal(string $date, string $locale = null): string
{
    global $LANG;

    $locale = $locale ?? $LANG['locale'] ?? 'en_US';

    $formatter = new IntlDateFormatter(
        $locale,
        IntlDateFormatter::MEDIUM,
        IntlDateFormatter::NONE
    );

    return $formatter->format(strtotime($date));
}

/**
 * Format datetime according to locale
 */
function formatDateTimeLocal(string $datetime, string $locale = null): string
{
    $locale = $locale ?? $LANG['locale'] ?? 'en_US';

    $formatter = new IntlDateFormatter(
        $locale,
        IntlDateFormatter::MEDIUM,
        IntlDateFormatter::SHORT
    );

    return $formatter->format(strtotime($datetime));
}

/**
 * Relative time formatting
 */
function formatRelativeTime(\DateTimeInterface $date, string $locale = null): string
{
    $locale = $locale ?? 'en_US';
    $now = new \DateTime();
    $diff = $now->diff($date);

    $formatter = new IntlRelativeTimeFormatter($locale);

    if ($diff->y > 0) {
        return $formatter->format($diff, 'year');
    }
    if ($diff->m > 0) {
        return $formatter->format($diff, 'month');
    }
    if ($diff->d > 0) {
        return $formatter->format($diff, 'day');
    }
    if ($diff->h > 0) {
        return $formatter->format($diff, 'hour');
    }

    return $formatter->format($diff, 'minute');
}
```

### Date Templates

```smarty
{* Using localized dates *}
<p>Created: {$service.order_date|date_format}</p>
<p>Next due: {$service.next_due_date|date_format:"%B %d, %Y"}</p>

{* Relative dates *}
<p>{$last_login|time_ago} ago</p>

{* Custom date format *}
{$invoice.date|date_format:"%d.%m.%Y"}
```

## Translation Workflow

### Translation File Creation

```bash
#!/bin/bash
# scripts/create_translation.sh

# Create new translation file from English base
SOURCE_FILE="lang/english.php"
TARGET_FILE="lang/deutsch.php"

# Extract translatable strings
grep -oP "(?<=_LANG\[')[^']+(?='\])" "$SOURCE_FILE" > /tmp/keys.txt

# Create translation template
cat > "lang/deutsch.php" << 'EOF'
<?php
/**
 * WHMCS Language File
 * Language: German
 * Translated from: English
 */

$_LANG = [
EOF

# Add entries (translator fills in values)
while read key; do
    echo "    '$key' => ''," >> "lang/deutsch.php"
done < /tmp/keys.txt

echo '];' >> "lang/deutsch.php"
```

### Translation Update Script

```php
<?php
// scripts/update_translations.php

/**
 * Update translation files with new strings
 */

function updateTranslationFile(string $baseFile, string $targetFile): void
{
    // Load base (English) strings
    require_once $baseFile;
    $baseStrings = $_LANG;

    // Load existing translations
    $translations = [];
    if (file_exists($targetFile)) {
        require_once $targetFile;
        $translations = $_LANG;
    }

    // Find new strings
    $newStrings = array_diff_key($baseStrings, $translations);

    // Update translations array
    $updated = array_merge($translations, $newStrings);

    // Write updated file
    $output = "<?php\n";
    $output .= "/**\n";
    $output .= " * Updated: " . date('Y-m-d') . "\n";
    $output .= " */\n";
    $output .= "\n\$_LANG = [\n";

    foreach ($updated as $key => $value) {
        $escaped = str_replace("'", "\\'", $value);
        $output .= "    '$key' => '$escaped',\n";
    }

    $output .= "];\n";

    file_put_contents($targetFile, $output);
}

// Run
updateTranslationFile('lang/english.php', 'lang/deutsch.php');
```

## Client Language Selection

### Language Selector Hook

```php
// includes/hooks/language_selector.php

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    // Get available languages
    $languages = \WHMCS\Language\Language::getLanguages();

    $currentLang = \WHMCS\Language\Language::getCurrentLanguage();

    // Build language selector HTML
    $html = '<div class="language-selector">';
    $html .= '<select id="language-selector" onchange="changeLanguage(this.value)">';

    foreach ($languages as $code => $name) {
        $selected = ($code === $currentLang) ? 'selected' : '';
        $html .= '<option value="' . $code . '" ' . $selected . '>' . $name . '</option>';
    }

    $html .= '</select></div>';

    $html .= '<script>
        function changeLanguage(lang) {
            jQuery.post("' . $_SERVER['PHP_SELF'] . '", {
                action: "setLanguage",
                language: lang,
                token: "' . generate_token('plain') . '"
            }).then(function() {
                location.reload();
            });
        }
    </script>';

    return $html;
});
```

## Best Practices

1. **Never Modify Core Files**: Use hooks to extend language strings
2. **Use UTF-8 Encoding**: All language files must be UTF-8
3. **Escape Output**: Use htmlspecialchars() for all user content
4. **RTL Support**: Test layouts with RTL languages
5. **Date/Time Formats**: Use locale-aware formatting
6. **Currency Formats**: Use IntlNumberFormatter
7. **Pluralization**: Handle plural forms correctly
8. **Variable Placement**: Consider word order variations
9. **Keep Strings Short**: Avoid concatenation issues
10. **Document Placeholders**: Clear documentation for translators

## Related Documentation

- [Email Template Variables](email-template-variables.md)
- [Smarty Template Reference](smarty-template-reference.md)
- [Hooks Reference](hooks-reference.md)
