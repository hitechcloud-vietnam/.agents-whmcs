# WHMCS Client Export

## Overview

Client export in WHMCS allows administrators to export client data for backup, migration, reporting, or integration with external systems. Exports can be generated in various formats including CSV, JSON, and XML.

## Export Configuration

### Enable Export

**Admin: Tools > Export > Export Clients**

```php
[
    'allow_export' => true,
    'export_permissions' => ['admin', 'manager'],
    'max_records' => 10000,
    'allowed_formats' => ['csv', 'json', 'xml'],
    'include_pii' => true              // Personal information
]
```

## Export Options

### Full Export

```php
// Export all client data
[
    'type' => 'full',
    'include' => [
        'profile' => true,
        'addresses' => true,
        'contacts' => true,
        'services' => true,
        'invoices' => true,
        'tickets' => true,
        'notes' => true,
        'custom_fields' => true
    ]
]
```

### Profile Only Export

```php
// Export client profile data
[
    'type' => 'profile',
    'fields' => [
        'id',
        'firstname',
        'lastname',
        'email',
        'company',
        'address1',
        'city',
        'state',
        'postcode',
        'country',
        'phonenumber',
        'created_at',
        'group'
    ]
]
```

### Filtered Export

```php
// Export filtered results
[
    'filter' => [
        'group' => 'Premium',
        'status' => 'Active',
        'created_after' => '2024-01-01',
        'created_before' => '2024-05-31'
    ]
]
```

## Export Formats

### CSV Export

```csv
firstname,lastname,email,company,city,country,created_at,group
John,Doe,john@example.com,Acme Inc,New York,US,2024-01-15,Premium
Jane,Smith,jane@example.com,,Los Angeles,US,2024-02-20,Standard
```

### JSON Export

```json
{
  "export_date": "2024-05-15",
  "total_records": 100,
  "clients": [
    {
      "id": 123,
      "firstname": "John",
      "lastname": "Doe",
      "email": "john@example.com",
      "company": "Acme Inc",
      "created_at": "2024-01-15"
    }
  ]
}
```

### XML Export

```xml
<?xml version="1.0" encoding="UTF-8"?>
<clients>
  <client>
    <id>123</id>
    <firstname>John</firstname>
    <lastname>Doe</lastname>
    <email>john@example.com</email>
  </client>
</clients>
```

## Field Selection

### Choose Fields to Export

```php
// Custom field selection
[
    'fields' => [
        'id',
        'firstname',
        'lastname',
        'email',
        'companyname',
        'address1',
        'city',
        'state',
        'postcode',
        'country',
        'phonenumber',
        'created_at',
        'group_id',
        'group_name',
        'total_spent',
        'last_login'
    ]
]
```

### Include Related Data

```php
// Include related records
[
    'include_services' => true,
    'include_invoices' => false,
    'include_tickets' => false,
    'include_contacts' => true
]
```

## Filter Options

### Client Filters

```php
// Filter exported clients
[
    'status' => 'Active',              // Active, Inactive, Cancelled
    'group' => 'Premium',              // Client group
    'country' => 'US',                 // Country filter
    'created_after' => '2024-01-01',    // Created after date
    'created_before' => '2024-05-31',   // Created before date
    'has_services' => true,            // Only with active services
    'min_spent' => 1000                 // Minimum total spent
]
```

### Multiple Filters

```php
// Combine filters
[
    'filters' => [
        ['field' => 'group', 'operator' => 'in', 'value' => ['Premium', 'VIP']],
        ['field' => 'total_spent', 'operator' => '>=', 'value' => 1000]
    ]
]
```

## Custom Field Export

### Include Custom Fields

```php
// Include custom fields
[
    'include_custom_fields' => true,
    'custom_field_ids' => [1, 2, 3],
    'custom_field_format' => 'separate_columns'  // separate_columns, json
]
```

### Export Result

```csv
email,custom_client_id,tier,account_manager
john@example.com,123456,VIP,John Admin
jane@example.com,789012,Standard,Jane Admin
```

## Export Process

### Step 1: Configure Export

```php
// Select options
[
    'format' => 'csv',
    'fields' => ['firstname', 'lastname', 'email', 'company'],
    'filters' => [...],
    'include_custom_fields' => true
]
```

### Step 2: Generate Export

```php
// Generate export file
[
    'file_name' => 'clients_export_20240515.csv',
    'encoding' => 'UTF-8',
    'delimiter' => ',',
    'enclosure' => '"'
]
```

### Step 3: Download

```php
// Export ready
[
    'download_url' => '/admin/download.php?type=clients&id=abc123',
    'expires' => '2024-05-16',
    'file_size' => '1.2 MB'
]
```

## Scheduled Exports

### Automated Export

```php
// Schedule regular exports
[
    'schedule' => '0 2 * * 0',        // Weekly on Sunday at 2 AM
    'format' => 'csv',
    'filters' => ['status' => 'Active'],
    'destination' => [
        'type' => 'ftp',
        'host' => 'ftp.example.com',
        'path' => '/exports/clients/'
    ],
    'notify_on_complete' => true
]
```

### Email Export

```php
// Email export file
[
    'email_to' => 'admin@example.com',
    'attach_file' => true,
    'include_summary' => true
]
```

## Export API

### API Export Function

```php
// Export via API
$result = localAPI('ExportClients', [
    'format' => 'csv',
    'fields' => ['id', 'firstname', 'lastname', 'email'],
    'filters' => [
        'status' => 'Active'
    ]
]);

// Response
// Returns download URL or file content
```

## Data Privacy

### PII Handling

```php
// Handle personal information
[
    'include_pii' => true,
    'encrypt_export' => true,
    'password_protect' => true,
    'auto_delete_days' => 7
]
```

### Anonymized Export

```php
// Export without PII
[
    'anonymize' => true,
    'preserve_fields' => ['id', 'group', 'total_spent'],
    'anonymize_fields' => ['email', 'name', 'address', 'phone']
]
```

## Best Practices

1. **Filter appropriately**: Export only needed data
2. **Secure exports**: Protect exported files with passwords
3. **Regular backups**: Schedule automated exports
4. **Delete old exports**: Remove files after download
5. **Document exports**: Track what was exported and when

## Related Documentation

- [Client Import](./whmcs-client-import.md)
- [Client Export Format](./whmcs-client-export-format.md)
- [Data Privacy](./whmcs-data-privacy.md)
- [Client Custom Fields](./whmcs-client-custom-fields.md)