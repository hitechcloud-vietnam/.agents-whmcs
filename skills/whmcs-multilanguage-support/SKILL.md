# WHMCS Multi-Language Support Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for adding multi-language support to WHMCS modules.

## When to Use

- Creating modules for international markets
- Supporting Vietnamese and English
- Adding language switching

## Language Structure

```
modules/addons/{module}/lang/
├── english.php
├── vietnamese.php
└── chinese.php
```

## Language Files

```php
<?php
// lang/english.php
if (!defined("WHMCS")) { die("Direct access denied"); }

return [
    'module_name' => 'Module Name',

    // General
    'btn_save' => 'Save',
    'btn_cancel' => 'Cancel',
    'btn_delete' => 'Delete',
    'btn_edit' => 'Edit',

    // Status
    'status_active' => 'Active',
    'status_pending' => 'Pending',
    'status_suspended' => 'Suspended',
    'status_terminated' => 'Terminated',

    // Messages
    'msg_success' => 'Operation completed successfully',
    'msg_error' => 'An error occurred',
    'msg_confirm' => 'Are you sure?',

    // Settings
    'settings_title' => 'Settings',
    'settings_api_key' => 'API Key',
    'settings_webhook' => 'Webhook URL',
];
```

```php
<?php
// lang/vietnamese.php
if (!defined("WHMCS")) { die("Direct access denied"); }

return [
    'module_name' => 'Ten Module',

    // General
    'btn_save' => 'Luu',
    'btn_cancel' => 'Huy',
    'btn_delete' => 'Xoa',
    'btn_edit' => 'Sua',

    // Status
    'status_active' => 'Hoat dong',
    'status_pending' => 'Dang cho',
    'status_suspended' => 'Bi tam hoan',
    'status_terminated' => 'Da ket thuc',

    // Messages
    'msg_success' => 'Thao tac thanh cong',
    'msg_error' => 'Da xay ra loi',
    'msg_confirm' => 'Ban co chac chan?',

    // Settings
    'settings_title' => 'Cai dat',
    'settings_api_key' => 'API Key',
    'settings_webhook' => 'Webhook URL',
];
```

## Using Language Strings

```php
function getLang(string $key, string $default = ''): string {
    global $LANG;

    $langFile = __DIR__ . '/lang/' . ($LANG['language'] ?? 'english') . '.php';

    if (file_exists($langFile)) {
        $strings = require $langFile;
        return $strings[$key] ?? $default;
    }

    // Default to English
    $strings = require __DIR__ . '/lang/english.php';
    return $strings[$key] ?? $default;
}
```

## Template Integration

```smarty
{* In templates *}
<button type="submit">{$LANG.btn_save}</button>

<span class="status-{$item.status|lower}">{$LANG['status_' . $item.status]}</span>
```

## Checklist

- [ ] English language file
- [ ] Vietnamese language file
- [ ] Language switching logic
- [ ] Fallback to English
- [ ] Date/time formatting by locale

---

**Related Skills:**
- whmcs-template-styling
- whmcs-addon-builder
- whmcs-internationalization