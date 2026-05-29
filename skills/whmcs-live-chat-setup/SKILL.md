# WHMCS Live Chat Setup

## Concept
Real-time chat integration for customer support.

## Code
```php
<?php
add_hook("ClientAreaPage", 1, function($vars) {
    return ["chat_enabled" => isModuleEnabled("livechat")];
});
```
