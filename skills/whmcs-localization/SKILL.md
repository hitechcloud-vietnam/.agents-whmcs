# WHMCS Module Localization Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing localization in WHMCS modules.

## When to Use

- Multi-language modules
- Regional settings
- Currency localization

## Localization Patterns

### Language File
```php
<?php
// modules/addons/{module}/lang/english.php

$_LANG['{module}_name'] = 'Module Name';
$_LANG['{module}_welcome'] = 'Welcome to {Module}';
$_LANG['{module}_error'] = 'An error occurred: {error}';
```

### Using Translations
```php
function {module}_output(array $vars): void {
    $lang = $_SESSION['Language'] ?? 'english';
    $translations = require __DIR__ . "/lang/{$lang}.php";

    echo '<h1>' . sprintf($translations['{module}_welcome'], $moduleName) . '</h1>';
}
```

### Get Translation
```php
function getTranslation(string $key, array $params = [], string $lang = 'english'): string {
    static $cache = [];

    if (!isset($cache[$lang])) {
        $cache[$lang] = require __DIR__ . "/lang/{$lang}.php";
    }

    $text = $cache[$lang][$key] ?? $key;

    if (empty($params)) {
        return $text;
    }

    return vsprintf($text, $params);
}
```

---

**Related Skills:**
- whmcs-multilanguage-support
- whmcs-internationalization
