# WHMCS Email Template Design Workflow

## Overview
Create and customize professional email templates for all WHMCS communications.

## Prerequisites
- WHMCS v8.0+
- Email template access

## Step-by-Step Guide

### Step 1: Email Template Configuration
```php
<?php
// Create custom email template
function create_email_template(array $templateData): int
{
    return \WHMCS\Database\Capsule::table('tblemailtemplates')->insertGetId([
        'name' => $templateData['name'],
        'subject' => $templateData['subject'],
        'message' => $templateData['body'],
        'type' => 'general',
        'custom' => 1,
        'disabled' => 0,
        'language' => '',
    ]);
}
```

### Step 2: HTML Email Template
```php
<?php
$template = '
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .header { background: #007bff; color: white; padding: 20px; }
        .content { padding: 20px; }
        .footer { background: #f8f9fa; padding: 10px; text-align: center; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Your Company Name</h1>
    </div>
    <div class="content">
        Dear {$client_name},<br><br>
        {$message}<br><br>
        {$signature}
    </div>
    <div class="footer">
        <p>Your Company Inc. | contact@example.com | +1 234 567 890</p>
    </div>
</body>
</html>';
```

## Checklist
- Email templates created
- HTML/CSS validated
- Variables substituted correctly
- Testing completed
