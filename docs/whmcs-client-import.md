# WHMCS Client Import

## Overview

Client import in WHMCS allows administrators to bulk import client data from CSV files or other sources. This is essential for migrating from other systems, initializing a new WHMCS installation, or syncing external databases.

## Import Configuration

### Enable Import

**Admin: Tools > Import > Import Clients**

```php
[
    'allow_import' => true,
    'import_permissions' => ['admin', 'manager'],
    'max_file_size' => 10,              // MB
    'allowed_formats' => ['csv', 'xlsx'],
    'encoding' => 'UTF-8'
]
```

## Import File Format

### CSV Structure

```csv
firstname,lastname,email,company,address1,address2,city,state,postcode,country,phonenumber,password
John,Doe,john@example.com,Acme Inc,123 Main St,,New York,NY,10001,US,555-0100,SecurePass123!
Jane,Smith,jane@example.com,,456 Oak Ave,Suite 100,Los Angeles,CA,90001,US,555-0200,SecurePass456!
```

### Field Mapping

| CSV Field | WHMCS Field | Required |
|-----------|-------------|----------|
| firstname | firstname | Yes |
| lastname | lastname | Yes |
| email | email | Yes |
| password | password | No |
| companyname | companyname | No |
| address1 | address1 | No |
| city | city | No |
| state | state | No |
| postcode | postcode | No |
| country | country | No |
| phonenumber | phonenumber | No |

## Import Process

### Step 1: Upload File

```php
// File upload
[
    'file' => 'clients_import.csv',
    'encoding' => 'UTF-8',
    'delimiter' => ',',
    'enclosure' => '"'
]
```

### Step 2: Field Mapping

```php
// Map CSV columns to WHMCS fields
[
    'column_a' => 'firstname',
    'column_b' => 'lastname',
    'column_c' => 'email',
    'column_d' => 'company',
    'column_e' => 'address1',
    'column_f' => 'city',
    'column_g' => 'state',
    'column_h' => 'postcode',
    'column_i' => 'country',
    'column_j' => 'phonenumber'
]
```

### Step 3: Validation

```php
// Validate import data
[
    'check_required_fields' => true,
    'check_email_format' => true,
    'check_email_unique' => true,
    'check_country_valid' => true,
    'check_duplicate_emails' => true
]
```

### Step 4: Preview

```php
// Preview first 10 records
[
    ['row' => 1, 'status' => 'valid', 'data' => [...]],
    ['row' => 2, 'status' => 'warning', 'message' => 'Email already exists'],
    ['row' => 3, 'status' => 'valid', 'data' => [...]]
]
```

### Step 5: Import

```php
// Import execution
[
    'total_rows' => 100,
    'valid_rows' => 95,
    'skipped_rows' => 5,
    'created_clients' => 95,
    'errors' => []
]
```

## Duplicate Handling

### Email Conflicts

```php
// Handle existing emails
[
    'on_duplicate' => 'skip',            // skip, update, merge
    'skip_duplicate_emails' => true,
    'log_skipped' => true
]
```

### Duplicate Resolution

```php
// Update existing clients
[
    'action' => 'update_existing',
    'match_by' => 'email',
    'update_fields' => ['address', 'phone', 'company'],
    'preserve_existing' => ['password']
]
```

## Custom Field Import

### Custom Fields Data

```csv
email,custom_field_1,custom_field_2,custom_field_3
john@example.com,CompanyA,123456,VIP
jane@example.com,CompanyB,789012,Standard
```

### Field Mapping

```php
// Map custom fields
[
    'custom_field_1' => 'company_type',
    'custom_field_2' => 'client_code',
    'custom_field_3' => 'tier'
]
```

## Password Handling

### Password Options

```php
// Import passwords
[
    'import_passwords' => true,
    'password_hash' => 'bcrypt',
    'generate_if_missing' => true,
    'send_welcome_email' => true,
    'require_change' => true
]
```

### Password Generation

```php
// Generate passwords for missing
[
    'auto_generate' => true,
    'password_length' => 12,
    'include_special' => true,
    'send_to_email' => true
]
```

## Validation Rules

### Data Validation

```php
// Validation rules
[
    'email' => [
        'required' => true,
        'format' => 'email',
        'unique' => true,
        'max_length' => 255
    ],
    'firstname' => [
        'required' => true,
        'max_length' => 100
    ],
    'country' => [
        'valid_values' => ['US', 'UK', 'CA', ...]
    ]
]
```

### Error Handling

```php
// Handle validation errors
[
    'stop_on_error' => false,
    'skip_invalid_rows' => true,
    'log_errors' => true,
    'error_threshold' => 10              // Stop after 10 errors
]
```

## Group Assignment

### Import with Groups

```csv
email,client_group,discount
john@example.com,Premium,10
jane@example.com,Standard,0
```

```php
// Group assignment
[
    'auto_create_groups' => true,
    'default_group' => 'Default',
    'group_field' => 'client_group'
]
```

## Import Results

### Success Report

```php
// Import summary
[
    'total_rows' => 100,
    'successful' => 95,
    'skipped' => 3,
    'errors' => 2,
    'created_groups' => 2,
    'import_duration' => '2 minutes'
]
```

### Error Report

```php
// Detailed errors
[
    ['row' => 15, 'email' => 'invalid-email', 'error' => 'Invalid email format'],
    ['row' => 42, 'email' => 'duplicate@example.com', 'error' => 'Email already exists']
]
```

## API Import

### API Import Function

```php
// Import via API
$result = localAPI('ImportClients', [
    'data' => [
        ['firstname' => 'John', 'lastname' => 'Doe', 'email' => 'john@example.com'],
        ['firstname' => 'Jane', 'lastname' => 'Smith', 'email' => 'jane@example.com']
    ],
    'options' => [
        'create_groups' => true,
        'send_welcome' => true
    ]
]);
```

## Scheduled Import

### Automated Import

```php
// FTP/SFTP scheduled import
[
    'enabled' => true,
    'schedule' => '0 2 * * *',         // Daily at 2 AM
    'ftp_server' => 'ftp.example.com',
    'ftp_path' => '/import/clients.csv',
    'delete_after_import' => false,
    'notify_on_complete' => true
]
```

## Best Practices

1. **Clean data first**: Verify CSV data before import
2. **Backup database**: Create backup before bulk import
3. **Test with small batch**: Test import with 5-10 records first
4. **Use UTF-8 encoding**: Ensure proper character encoding
5. **Review errors**: Check and fix import errors promptly

## Related Documentation

- [Client Export](./whmcs-client-export.md)
- [Client Creation](./whmcs-client-creation.md)
- [Client Groups](./whmcs-client-groups.md)
- [Client Custom Fields](./whmcs-client-custom-fields.md)