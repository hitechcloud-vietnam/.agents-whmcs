# WHMCS Custom Fields Configuration Workflow

## Overview
Comprehensive workflow for configuring and managing custom fields for clients and services.

## Prerequisites
- WHMCS v8.0+
- Admin access

## Step-by-Step Guide

### Step 1: Create Custom Field
```php
<?php
function create_custom_field(array $fieldConfig): int
{
    return \WHMCS\Database\Capsule::table('tblcustomfields')->insertGetId([
        'type' => $fieldConfig['type'],
        'relid' => $fieldConfig['relid'] ?? 0,
        'fieldname' => $fieldConfig['fieldname'],
        'fieldtype' => $fieldConfig['fieldtype'],
        'description' => $fieldConfig['description'] ?? '',
        'fieldoptions' => $fieldConfig['fieldoptions'] ?? '',
        'regexplabel' => $fieldConfig['regexplabel'] ?? '',
        'adminonly' => $fieldConfig['adminonly'] ?? '',
        'required' => $fieldConfig['required'] ?? 0,
        'showinvoice' => $fieldConfig['showinvoice'] ?? 0,
        'sortorder' => $fieldConfig['sortorder'] ?? 0,
    ]);
}
```

### Step 2: Custom Field Types
```php
$fieldTypes = [
    'text' => 'Text String',
    'textarea' => 'Text Area',
    'password' => 'Password',
    'dropdown' => 'Drop Down',
    'checkbox' => 'Check Box',
    'radio' => 'Radio Button',
    'date' => 'Date',
];
```

## Checklist
- Client custom fields created
- Product custom fields configured
- Validation rules set
- Display options configured
