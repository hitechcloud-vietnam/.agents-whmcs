# WHMCS White Label Setup Workflow

## Overview
Configure WHMCS for white label deployment with custom branding and domain.

## Prerequisites
- WHMCS v8.0+
- Reseller access

## Step-by-Step Guide

### Step 1: White Label Configuration
```php
<?php
// configuration.php additions
define('WHITELABEL_MODE', true);
define('COMPANY_NAME', 'Your Brand Name');
define('COMPANY_URL', 'https://yourdomain.com');
define('COMPANY_EMAIL', 'support@yourdomain.com');
define('COMPANY_LOGO', '/path/to/logo.png');
```

### Step 2: Branding Hooks
```php
<?php
add_hook('ClientAreaPage', 1, function($vars) {
    return [
        'company_name' => 'Your Brand Name',
        'company_url' => 'https://yourdomain.com',
        'support_email' => 'support@yourdomain.com',
        'custom_css' => '/css/whitelabel.css',
    ];
});
```

### Step 3: Remove WHMCS Branding
```php
<?php
add_hook('OutputLoadTemplate', 1, function($vars) {
    // Remove powered by WHMCS
    return ['hide_powered_by' => true];
});
```

## Checklist
- Company details configured
- Logo uploaded
- Custom styling applied
- Branded emails created
