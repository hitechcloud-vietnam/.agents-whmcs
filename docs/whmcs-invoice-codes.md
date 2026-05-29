# WHMCS Invoice Number Formats Documentation

## Overview

Invoice numbering in WHMCS is fully customizable, allowing you to define formats that meet your business requirements, regulatory needs, and branding preferences.

## Configuration

### Invoice Numbering Settings

Navigate to: **Configuration > General Settings > Invoices**

```php
// Invoice Numbering Configuration
$invoiceNumberConfig = [
    // Format Settings
    'format' => '{prefix}{year}{month}{sequence}',
    'prefix' => 'INV-',
    'separator' => '',
    'sequence_length' => 4,
    'sequence_start' => 1,
    'sequence_padding' => true,         // Pad with zeros

    // Reset Settings
    'reset_frequency' => 'never',        // never, yearly, monthly
    'reset_date' => '01-01',            // For yearly reset

    // Prefix Options
    'prefix_by_product' => false,
    'prefix_by_client' => false,

    // Suffix Options
    'suffix_type' => 'none',            // none, client, product
    'suffix_length' => 3
];
```

## Format Tokens

### Available Tokens

| Token | Description | Example |
|-------|-------------|---------|
| `{prefix}` | Custom prefix | INV- |
| `{year}` | 4-digit year | 2024 |
| `{year2}` | 2-digit year | 24 |
| `{month}` | 2-digit month | 01 |
| `{month_name}` | Month name | January |
| `{quarter}` | Quarter (1-4) | Q1 |
| `{day}` | Day of month | 15 |
| `{sequence}` | Sequential number | 0001 |
| `{seq}` | Sequential without padding | 1 |
| `{client_id}` | Client ID | 12345 |
| `{client_gen}` | Generated client code | C12345 |
| `{product_id}` | Product ID | 67890 |
| `{user_id}` | User ID | 12345 |

## Format Examples

### Standard Format

```
Format: INV-{year}{month}{sequence}
Result: INV-2024010001

Format: {prefix}{year}{month}{sequence}
Result: INV-2024010001
```

### With Separator

```
Format: {prefix}/{year}/{month}/{sequence}
Result: INV/2024/01/0001

Format: {prefix}-{year}-{month}-{sequence}
Result: INV-2024-01-0001
```

### Client-Based

```
Format: {prefix}{client_id}-{sequence}
Result: INV-12345-0001

Format: C{client_id}-{year}{sequence}
Result: C12345-240001
```

### Sequential Without Reset

```
Format: {prefix}{sequence}
Result: INV-0000001

Format: {prefix}{year}{seq}
Result: INV-20240000001
```

### With Quarter

```
Format: {prefix}{year}Q{quarter}{sequence}
Result: INV-2024Q10001
```

## Number Series Management

### Create Number Series

```php
// Define multiple number series
$numberSeries = [
    'default' => [
        'prefix' => 'INV-',
        'format' => '{prefix}{year}{month}{sequence}',
        'sequence' => 1,
        'reset_frequency' => 'monthly'
    ],
    'recurring' => [
        'prefix' => 'REC-',
        'format' => '{prefix}{year}{sequence}',
        'sequence' => 1,
        'reset_frequency' => 'yearly'
    ],
    'proforma' => [
        'prefix' => 'PRO-',
        'format' => '{prefix}{year}{month}{sequence}',
        'sequence' => 1,
        'reset_frequency' => 'monthly'
    ],
    'credit_note' => [
        'prefix' => 'CR-',
        'format' => '{prefix}{year}{sequence}',
        'sequence' => 1,
        'reset_frequency' => 'yearly'
    ]
];
```

### Series Assignment

```php
// Auto-assign series based on type
$seriesAssignment = [
    'invoice' => 'default',
    'recurring_invoice' => 'recurring',
    'proforma_invoice' => 'proforma',
    'credit_note' => 'credit_note',
    'refund' => 'refund'
];
```

## Custom Numbering Rules

### By Product/Service Type

```php
// Product-specific numbering
$productPrefix = [
    'hosting' => [
        'prefix' => 'HOST-',
        'example' => 'HOST-20240001'
    ],
    'domain' => [
        'prefix' => 'DOM-',
        'example' => 'DOM-20240001'
    ],
    'ssl' => [
        'prefix' => 'SSL-',
        'example' => 'SSL-20240001'
    ],
    'addon' => [
        'prefix' => 'ADD-',
        'example' => 'ADD-20240001'
    ]
];
```

### By Client/Tags

```php
// Client-based prefix
$clientPrefix = [
    'enabled' => true,
    'by_tier' => [
        'standard' => '',
        'premium' => 'P-',
        'enterprise' => 'E-'
    ],
    'by_reseller' => [
        'enabled' => true,
        'prefix' => 'R-'
    ]
];
```

## Invoice Type Formats

### Regular Invoice

```
Format: INV-{year}{month}{sequence}
Example: INV-2024010001
```

### Recurring Invoice

```
Format: REC-{year}{month}{sequence}
Example: REC-2024010001
```

### Proforma Invoice

```
Format: PRO-{year}{month}{sequence}
Example: PRO-2024010001
```

### Credit Note

```
Format: CN-{year}{sequence}
Example: CN-20240001
```

### Refund

```
Format: REF-{year}{sequence}
Example: REF-20240001
```

## API Reference

### Update Invoice Number

```http
PATCH /billing/invoices/{invoice_id}
```

**Request Body:**

```json
{
  "invoice_number": "INV-2024010005",
  "reason": "Manual adjustment"
}
```

### Generate Next Number

```http
GET /billing/invoices/next-number
```

**Query Parameters:**
- `type`: invoice, recurring, proforma, credit_note

**Response:**

```json
{
  "type": "invoice",
  "next_number": "INV-2024010002",
  "sequence": 2
}
```

### Reserve Number

```http
POST /billing/invoices/reserve-number
```

**Request Body:**

```json
{
  "type": "invoice",
  "quantity": 10
}
```

## Number Formatting

### Sequence Padding

```php
// Sequence padding configuration
$sequenceConfig = [
    'padding_enabled' => true,
    'padding_length' => 4,
    'padding_character' => '0',

    // Examples:
    // padding_length=4: 0001, 0002, ... 9999
    // padding_length=5: 00001, 00002, ... 99999
    // padding_length=6: 000001, 000002, ... 999999
];
```

### Separator Characters

```php
// Separator options
$separators = [
    'none' => 'INV2024010001',
    'dash' => 'INV-2024-01-0001',
    'slash' => 'INV/2024/01/0001',
    'dot' => 'INV.2024.01.0001',
    'space' => 'INV 2024 01 0001'
];
```

## Validation

### Number Validation Rules

```php
// Validation configuration
$validationConfig = [
    'required' => true,
    'unique' => true,
    'min_length' => 5,
    'max_length' => 50,
    'pattern' => '/^[A-Z]{0,5}-?\d{4,}$/',

    // Block duplicates
    'duplicate_check' => true,
    'allow_override' => false,           // Allow manual override

    // Reserved patterns
    'reserved_patterns' => ['TEST', 'VOID', 'CANCEL']
];
```

### Validation API

```http
POST /billing/invoices/validate-number
```

**Request Body:**

```json
{
  "invoice_number": "INV-2024010001",
  "type": "invoice"
}
```

**Response:**

```json
{
  "valid": true,
  "suggestions": []
}
```

**Error Response:**

```json
{
  "valid": false,
  "errors": [
    "Invoice number already exists",
    "Invalid format pattern"
  ],
  "suggestions": [
    "INV-2024010002",
    "INV-2024010003"
  ]
}
```

## Number Series Operations

### Reset Series

```http
POST /billing/invoices/series/reset
```

**Request Body:**

```json
{
  "series": "default",
  "reset_to": 1,
  "confirm": true
}
```

### Get Series Info

```http
GET /billing/invoices/series/{series_name}
```

**Response:**

```json
{
  "series": "default",
  "current_sequence": 150,
  "last_number": "INV-2024010150",
  "created_at": "2024-01-01T00:00:00Z",
  "last_used": "2024-01-15T10:30:00Z"
}
```

## Renumbering Invoices

### Bulk Renumber

```http
POST /billing/invoices/renumber
```

**Request Body:**

```json
{
  "from_date": "2024-01-01",
  "to_date": "2024-01-31",
  "new_format": "INV-{year}{month}{sequence}",
  "start_sequence": 1,
  "preview_only": false
}
```

**Response:**

```json
{
  "preview": true,
  "changes": [
    {"old_number" => "1001", "new_number" => "INV-2024010001"},
    {"old_number" => "1002", "new_number" => "INV-2024010002"}
  ],
  "total_changes": 50
}
```

## Display Settings

### Invoice Number Display

```php
// Display settings
$displayConfig = [
    'show_on_pdf' => true,
    'show_on_email' => true,
    'show_on_portal' => true,
    'label' => 'Invoice Number',
    'format_on_display' => '{prefix}{sequence}',    // Short format for display
    'format_on_pdf' => '{prefix}{year}{month}{sequence}'  // Full format
];
```

## Best Practices

1. **Be consistent** - Use the same format across all invoice types
2. **Include year/month** - Makes filtering and reporting easier
3. **Plan for growth** - Allow enough sequence digits
4. **Follow regulations** - Some countries have specific requirements
5. **Avoid confusion** - Don't use similar formats for different types
6. **Document format** - Keep documentation of your numbering system

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Duplicate numbers | Manual override error | Check sequence integrity |
| Gaps in sequence | Failed transactions | Track reserved numbers |
| Wrong format | Format misconfiguration | Verify format string |
| Sequence not resetting | Reset settings wrong | Check reset frequency |

### Debug Commands

```bash
# Check next invoice number
whmcscli invoice next-number

# View series status
whmcscli invoice series-status --series=default

# Verify number uniqueness
whmcscli invoice verify-number --number=INV-2024010001

# Fix sequence
whmcscli invoice fix-sequence --series=default --new_sequence=1000
```

## See Also

- [Recurring Invoice](./whmcs-recurring-invoice.md)
- [Payment Terms](./whmcs-payment-terms.md)
- [Invoice Templates](../developer/invoice-templates.md)
