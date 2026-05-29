# WHMCS Client Custom Fields

## Overview

Client custom fields in WHMCS allow administrators to capture additional information beyond the standard client profile fields. Custom fields can be used for segmentation, automation, and data collection.

## Custom Field Configuration

### Accessing Custom Fields

**Admin: Configuration > Custom Fields > Client Custom Fields**

### Create Custom Field

```php
// New custom field
[
    'field_name' => 'client_type',
    'field_label' => 'Client Type',
    'field_type' => 'dropdown',
    'required' => false,
    'show_in_client_area' => true,
    'show_in_admin' => true,
    'show_on_invoice' => false
]
```

## Field Types

### Text Field

```php
// Text input
[
    'type' => 'text',
    'max_length' => 255,
    'placeholder' => 'Enter value',
    'validation' => 'alphanumeric'
]
```

### Textarea

```php
// Multi-line text
[
    'type' => 'textarea',
    'max_length' => 5000,
    'rows' => 4
]
```

### Dropdown

```php
// Select from options
[
    'type' => 'dropdown',
    'options' => [
        'Individual',
        'Small Business',
        'Enterprise'
    ],
    'allow_other' => true
]
```

### Checkbox

```php
// Boolean field
[
    'type' => 'checkbox',
    'description' => 'Enable marketing emails'
]
```

### Radio Buttons

```php
// Single selection
[
    'type' => 'radio',
    'options' => [
        'Newsletter',
        'SMS',
        'Email Only'
    ]
]
```

### Date Field

```php
// Date picker
[
    'type' => 'date',
    'format' => 'YYYY-MM-DD',
    'min_date' => '2020-01-01',
    'max_date' => '2030-12-31'
]
```

### Number Field

```php
// Numeric input
[
    'type' => 'number',
    'min_value' => 0,
    'max_value' => 1000000,
    'decimals' => 0
]
```

## Field Configuration Options

### Display Options

```php
// Configure field display
[
    'field_label' => 'Account Manager',
    'description' => 'Assigned account manager',
    'tooltip' => 'Primary contact for this client',
    'field_order' => 10,
    'show_in_client_area' => true,
    'show_in_admin' => true,
    'show_on_invoice' => false,
    'show_on_quote' => false
]
```

### Validation Rules

```php
// Field validation
[
    'required' => false,
    'min_length' => 0,
    'max_length' => 255,
    'regex_pattern' => null,
    'unique' => false
]
```

## Custom Field Management

### Edit Field

```php
// Update custom field
[
    'field_id' => 1,
    'field_label' => 'Account Manager',
    'required' => true,
    'options' => ['Admin 1', 'Admin 2', 'Admin 3']
]
```

### Reorder Fields

```php
// Change field order
[
    'field_id' => 1,
    'sort_order' => 5
]
```

### Delete Field

```php
// Remove custom field
[
    'field_id' => 1,
    'preserve_data' => true,
    'migrate_to_field' => null
]
```

## Client Custom Field Values

### Set Field Value

**Admin: Clients > Select Client > Edit > Custom Fields**

```php
// Set field value
[
    'userid' => 123,
    'field_id' => 1,
    'value' => 'Enterprise',
    'updated_by' => 'admin_id',
    'updated_at' => '2024-05-15'
]
```

### Bulk Update

```php
// Update multiple clients
[
    'action' => 'bulk_update',
    'field_id' => 1,
    'value' => 'Enterprise',
    'filter' => ['total_spent >= 10000']
]
```

## Custom Field Display

### Admin View

```php
// Display in client profile
[
    'field_label' => 'Client Type',
    'field_value' => 'Enterprise',
    'show_edit' => true
]
```

### Client Area Display

```smarty
// Display in client area
{foreach $client.customfields as $field}
    <div class="custom-field">
        <label>{$field.label}</label>
        <span>{$field.value}</span>
    </div>
{/foreach}
```

### Invoice Display

```php
// Include on invoices
[
    'field_id' => 1,
    'show_on_invoice' => true,
    'invoice_label' => 'Customer Reference'
]
```

## Custom Field Integration

### Automation Integration

```php
// Use custom fields in automation
[
    'trigger' => 'client_created',
    'conditions' => [
        ['field' => 'client_type', 'value' => 'Enterprise']
    ],
    'actions' => [
        ['action' => 'assign_group', 'group' => 'Enterprise'],
        ['action' => 'send_email', 'template' => 'enterprise_welcome']
    ]
]
```

### Pricing Integration

```php
// Use in pricing
[
    'condition' => 'custom.client_type = "Enterprise"',
    'discount' => 15
]
```

## Validation

### Required Fields

```php
// Make field required
[
    'field_id' => 1,
    'required' => true,
    'required_error' => 'Please select a client type'
]
```

### Custom Validation

```php
// Custom validation rules
[
    'field_id' => 1,
    'validation_type' => 'regex',
    'regex_pattern' => '^[A-Z]{2}[0-9]{6}$',
    'validation_message' => 'Enter valid company code (e.g., US123456)'
]
```

## Search and Filter

### Search by Custom Field

```php
// Filter by custom field
[
    'field' => 'client_type',
    'operator' => 'equals',
    'value' => 'Enterprise'
]
```

### Report by Custom Field

```php
// Group report by custom field
[
    'group_by' => 'client_type',
    'metrics' => ['count', 'total_spent', 'avg_spent']
]
```

## API Functions

```php
// Get custom fields
$result = localAPI('GetClientCustomFields', [
    'clientid' => 123
]);

// Update custom field
$result = localAPI('UpdateClientCustomField', [
    'clientid' => 123,
    'fieldid' => 1,
    'value' => 'Enterprise'
]);

// Get custom field options
$result = localAPI('GetCustomFieldOptions', [
    'fieldid' => 1
]);
```

## Hooks

```php
// Hook: CustomFieldValueChanged
add_hook('CustomFieldValueChanged', 1, function($vars) {
    // $vars['userid']
    // $vars['fieldid']
    // $vars['old_value']
    // $vars['new_value']
});
```

## Best Practices

1. **Plan ahead**: Define needed fields before import
2. **Use appropriate types**: Choose right field type
3. **Clear labels**: Use descriptive field names
4. **Validate data**: Ensure data integrity
5. **Document fields**: Keep track of field purposes

## Related Documentation

- [Client Creation](./whmcs-client-creation.md)
- [Client Groups](./whmcs-client-groups.md)
- [Client Tags](./whmcs-client-tags.md)
- [Client Import](./whmcs-client-import.md)