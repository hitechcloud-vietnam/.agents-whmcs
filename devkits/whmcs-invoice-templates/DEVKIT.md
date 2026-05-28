# WHMCS Invoice Templates Module

Custom invoice template management with multiple templates and preview.

## Features

- Multiple invoice templates
- Template customization (logo, colors, layout)
- Template categories (standard, proforma, credit note, receipt)
- Live template preview
- Template variables support
- PDF generation settings
- Email template integration
- Template versioning
- Default template selection
- Template cloning
- Multi-language support
- Custom CSS/HTML editor

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/invoicetemplates/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure and create templates

## Usage

```php
// Create invoice template
$result = invoicetemplates_CreateTemplate(array(
    'template_name' => 'Modern Corporate',
    'template_key' => 'modern_corporate',
    'category' => 'standard',
    'header_html' => '<div class="invoice-header">...</div>',
    'body_html' => '<table>...</table>',
    'footer_html' => '<div class="invoice-footer">...</div>',
    'css' => '.invoice-header { background: #fff; }',
    'is_default' => true
));

// Get template
$template = invoicetemplates_GetTemplate($templateId);

// Get all templates
$templates = invoicetemplates_GetTemplates(array(
    'category' => 'standard',
    'is_active' => true
));

// Update template
invoicetemplates_UpdateTemplate($templateId, array(
    'header_html' => '<h1>Updated Header</h1>',
    'css' => '.invoice-header { background: #f5f5f5; }'
));

// Clone template
$cloned = invoicetemplates_CloneTemplate($templateId, 'My Copy');

// Delete template
invoicetemplates_DeleteTemplate($templateId);

// Set default template
invoicetemplates_SetDefault($templateId);

// Generate preview
$preview = invoicetemplates_GeneratePreview($templateId, $invoiceId);

// Get template variables
$variables = invoicetemplates_GetVariables();
// Returns: $client_name, $invoice_number, $invoice_date, etc.

// Get available placeholders (for editor)
$placeholders = invoicetemplates_GetPlaceholders();
// Returns: {client_name}, {invoice_number}, {invoice_date}, etc.

// Get template categories
$categories = invoicetemplates_GetCategories();

// Set template for invoice
invoicetemplates_SetInvoiceTemplate($invoiceId, $templateId);

// Get invoice template
$invoiceTemplate = invoicetemplates_GetInvoiceTemplate($invoiceId);

// Set template for client
invoicetemplates_SetClientTemplate($clientId, $templateId);

// Get client template
$clientTemplate = invoicetemplates_GetClientTemplate($clientId);

// Add template translation
invoicetemplates_AddTranslation($templateId, 'vi', array(
    'invoice_title' => 'Hoá đơn',
    'terms' => 'Điều khoản'
));

// Export template
$export = invoicetemplates_ExportTemplate($templateId);

// Import template
$import = invoicetemplates_ImportTemplate($exportData);

// Create template version
invoicetemplates_CreateVersion($templateId, 'Fixed alignment issues');
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultTemplate | dropdown | default | Default template |
| PDFEngine | dropdown | tcpdf | PDF generation engine |
| PaperSize | dropdown | A4 | Paper size |
| Orientation | dropdown | portrait | Page orientation |
| EnableTranslations | yesno | yes | Enable translations |
| CacheEnabled | yesno | yes | Cache compiled templates |
| LogoMaxWidth | text | 200 | Logo max width (px) |
| LogoMaxHeight | text | 100 | Logo max height (px) |

## Template Categories

| Category | Description |
|----------|-------------|
| standard | Standard invoices |
| proforma | Proforma invoices |
| credit_note | Credit notes |
| receipt | Payment receipts |
| dunning | Dunning notices |
| quote | Quote templates |
| customized | Custom templates |

## Template Variables

| Variable | Description |
|---------|-------------|
| $client_name | Client full name |
| $client_email | Client email |
| $client_address | Client address |
| $invoice_number | Invoice number |
| $invoice_date | Invoice date |
| $invoice_due_date | Due date |
| $invoice_subtotal | Subtotal |
| $invoice_tax | Tax amount |
| $invoice_total | Total amount |
| $invoice_currency | Currency |
| $payment_method | Payment method |
| $items | Line items |
| $notes | Invoice notes |
| $company_name | Company name |

## Database Tables

- `mod_invoicetemplates_templates` - Template definitions
- `mod_invoicetemplates_versions` - Version history
- `mod_invoicetemplates_translations` - Translations
- `mod_invoicetemplates_settings` - Template settings
- `mod_invoicetemplates_client_settings` - Per-client settings

## API Functions

| Function | Description |
|----------|-------------|
| `invoicetemplates_CreateTemplate()` | Create template |
| `invoicetemplates_GetTemplate()` | Get template details |
| `invoicetemplates_GetTemplates()` | List templates |
| `invoicetemplates_UpdateTemplate()` | Update template |
| `invoicetemplates_CloneTemplate()` | Clone template |
| `invoicetemplates_DeleteTemplate()` | Delete template |
| `invoicetemplates_SetDefault()` | Set default |
| `invoicetemplates_GeneratePreview()` | Generate preview HTML |
| `invoicetemplates_GetVariables()` | Get variables |
| `invoicetemplates_GetPlaceholders()` | Get placeholders |
| `invoicetemplates_GetCategories()` | Get categories |
| `invoicetemplates_SetInvoiceTemplate()` | Assign to invoice |
| `invoicetemplates_SetClientTemplate()` | Assign to client |
| `invoicetemplates_AddTranslation()` | Add translation |
| `invoicetemplates_ExportTemplate()` | Export JSON |
| `invoicetemplates_ImportTemplate()` | Import JSON |
| `invoicetemplates_CreateVersion()` | Create version snapshot |
