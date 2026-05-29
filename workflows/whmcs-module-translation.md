# WHMCS Module Translation Workflow

## Description
Create and manage translations for WHMCS modules.

## Steps

### Step 1: Create Translation File
```php
<?php
// languages/en.php

return [
    // Module info
    'module_name' => 'Module Name',
    'module_description' => 'Module description text',
    
    // Navigation
    'nav_dashboard' => 'Dashboard',
    'nav_settings' => 'Settings',
    'nav_reports' => 'Reports',
    'nav_help' => 'Help',
    
    // Actions
    'action_save' => 'Save',
    'action_cancel' => 'Cancel',
    'action_delete' => 'Delete',
    'action_edit' => 'Edit',
    'action_add' => 'Add',
    'action_search' => 'Search',
    
    // Messages
    'msg_saved' => 'Settings saved successfully.',
    'msg_error' => 'An error occurred.',
    'msg_confirm' => 'Are you sure?',
    'msg_loading' => 'Loading...',
    'msg_no_data' => 'No data found.',
    
    // Errors
    'err_required' => 'This field is required.',
    'err_invalid' => 'Invalid value.',
    'err_connection' => 'Connection failed.',
    
    // Settings
    'setting_api_key' => 'API Key',
    'setting_webhook' => 'Webhook URL',
    'setting_debug' => 'Debug Mode',
];
```

### Step 2: Create Vietnamese Translation
```php
<?php
// languages/vi.php

return [
    'module_name' => 'Tên Mô-đun',
    'module_description' => 'Mô tả mô-đun',
    
    'nav_dashboard' => 'Bảng điều khiển',
    'nav_settings' => 'Cài đặt',
    'nav_reports' => 'Báo cáo',
    'nav_help' => 'Trợ giúp',
    
    'action_save' => 'Lưu',
    'action_cancel' => 'Hủy',
    'action_delete' => 'Xóa',
    'action_edit' => 'Sửa',
    'action_add' => 'Thêm',
    'action_search' => 'Tìm kiếm',
    
    'msg_saved' => 'Cài đặt đã được lưu thành công.',
    'msg_error' => 'Đã xảy ra lỗi.',
    'msg_confirm' => 'Bạn có chắc chắn?',
    'msg_loading' => 'Đang tải...',
    'msg_no_data' => 'Không có dữ liệu.',
    
    'err_required' => 'Trường này bắt buộc.',
    'err_invalid' => 'Giá trị không hợp lệ.',
    'err_connection' => 'Kết nối thất bại.',
    
    'setting_api_key' => 'Khóa API',
    'setting_webhook' => 'URL Webhook',
    'setting_debug' => 'Chế độ gỡ lỗi',
];
```

### Step 3: Load Translation
```php
<?php
// In module file

function clicodes_example_loadLang($module)
{
    $lang = App::getFromRequest('language') ?: App::getAdminLanguage() ?: 'en';
    
    $langFile = __DIR__ . "/languages/{$lang}.php";
    
    if (!file_exists($langFile)) {
        $langFile = __DIR__ . '/languages/en.php';
    }
    
    return require $langFile;
}
```

### Step 4: Use in Templates
```smarty
<h3>{$LANG.module_name}</h3>

<button>{$LANG.action_save}</button>

{if $error}
    <div class="alert">{$LANG.err_required}</div>
{/if}
```

## Common Language Codes
| Code | Language |
|------|----------|
| en | English |
| vi | Vietnamese |
| zh-cn | Chinese (Simplified) |
| zh-tw | Chinese (Traditional) |
| ja | Japanese |
| ko | Korean |
| fr | French |
| de | German |
| es | Spanish |
| pt-br | Portuguese (Brazil) |

## Translation Tips
- Use placeholders: `%s`, `%d`
- Escape HTML entities
- Keep strings short
- Contextual comments for translators

## Tags
- translation
- i18n
- localization
- language