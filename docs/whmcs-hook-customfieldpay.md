# WHMCS CustomFieldSave Hook Reference

## Overview

The `CustomFieldSave` hook fires when custom field values are saved for a client or service. This hook triggers during custom field updates and allows validation, transformation, and integration.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `fieldid` | int | The custom field ID |
| `fieldname` | string | The custom field name |
| `fieldtype` | string | Field type (text, dropdown, etc.) |
| `relid` | int | Related entity ID (client/service ID) |
| `value` | string | The field value |
| `oldvalue` | string | Previous value |
| `type` | string | Entity type (client, service, domain, etc.) |

## Example Implementation

```php
<?php
add_hook('CustomFieldSave', 1, function(array $params) {
    // Log custom field change
    logActivity("Custom field '{$params['fieldname']}' updated for entity {$params['relid']}");
    
    return $params;
});
```

## Validation

```php
<?php
add_hook('CustomFieldSave', 1, function(array $params) {
    // 1. Validate phone number format
    if ($params['fieldname'] === 'phone_number') {
        $pattern = '/^\+?[1-9]\d{1,14}$/';
        if (!preg_match($pattern, $params['value'])) {
            return [
                'success' => false,
                'errorMessage' => 'Invalid phone number format. Use E.164 format.'
            ];
        }
    }
    
    // 2. Validate tax ID
    if ($params['fieldname'] === 'tax_id') {
        if (!validateTaxId($params['value'])) {
            return [
                'success' => false,
                'errorMessage' => 'Invalid tax identification number'
            ];
        }
    }
    
    // 3. Limit text field length
    if ($params['fieldtype'] === 'text' && strlen($params['value']) > 255) {
        return [
            'success' => false,
            'errorMessage' => 'Field value exceeds maximum length'
        ];
    }
    
    return $params;
});
```

## Data Transformation

```php
<?php
add_hook('CustomFieldSave', 1, function(array $params) {
    // 1. Normalize company name
    if ($params['fieldname'] === 'company_name') {
        $normalized = normalizeCompanyName($params['value']);
        $params['value'] = $normalized;
    }
    
    // 2. Format date fields
    if ($params['fieldtype'] === 'date') {
        $params['value'] = formatDateForStorage($params['value']);
    }
    
    // 3. Sanitize text input
    if ($params['fieldtype'] === 'text') {
        $params['value'] = sanitizeTextInput($params['value']);
    }
    
    // 4. Uppercase country codes
    if ($params['fieldname'] === 'country_code') {
        $params['value'] = strtoupper(trim($params['value']));
    }
    
    return $params;
});
```

## External Integration

```php
<?php
add_hook('CustomFieldSave', 1, function(array $params) {
    // 1. Sync to CRM
    if ($params['type'] === 'client') {
        syncClientFieldToCRM($params['relid'], $params['fieldname'], $params['value']);
    }
    
    // 2. Trigger webhooks for specific fields
    $webhookFields = ['vip_status', 'account_type', 'tier_level'];
    if (in_array($params['fieldname'], $webhookFields)) {
        sendWebhook('custom_field.updated', [
            'entity_type' => $params['type'],
            'entity_id' => $params['relid'],
            'field' => $params['fieldname'],
            'old_value' => $params['oldvalue'],
            'new_value' => $params['value']
        ]);
    }
    
    // 3. Update related records
    if ($params['fieldname'] === 'primary_contact') {
        updateRelatedRecords($params['relid'], $params['value']);
    }
    
    return $params;
});
```

## Conditional Logic

```php
<?php
add_hook('CustomFieldSave', 1, function(array $params) {
    // 1. Auto-populate related fields
    if ($params['fieldname'] === 'vat_number') {
        $country = extractCountryFromVAT($params['value']);
        if ($country) {
            updateCustomField($params['relid'], 'detected_country', $country);
        }
    }
    
    // 2. Set default values for other fields
    if ($params['fieldname'] === 'industry' && $params['type'] === 'client') {
        $defaultPreferences = getIndustryDefaults($params['value']);
        foreach ($defaultPreferences as $field => $value) {
            updateCustomField($params['relid'], $field, $value);
        }
    }
    
    // 3. Trigger notifications for specific values
    if ($params['fieldname'] === 'risk_level' && $params['value'] === 'high') {
        sendAdminNotification('security', [
            'subject' => 'High Risk Client Flagged',
            'message' => "Client {$params['relid']} marked as high risk"
        ]);
    }
    
    return $params;
});
```

## Use Cases

- **Validation**: Validate input formats
- **Transformation**: Normalize and format values
- **Integration**: Sync with external systems
- **Conditional Logic**: Auto-populate related fields
- **Audit Trail**: Log changes to sensitive fields

## Notes

- Runs when custom fields are saved
- Can modify value before storage
- Can return error to prevent save
- Different entity types available (client, service, domain)

## Related Hooks

- `ClientEdit` - Client profile editing
- `ServiceCreate` - Service creation
- `CustomFieldDisplay` - Display customization

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Custom Fields Configuration](../whmcs-custom-fields.md)